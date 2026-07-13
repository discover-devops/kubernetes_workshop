# Lab 1: Horizontal Pod Autoscaler (HPA) on Amazon EKS

**Level:** Beginner–Intermediate
**Duration:** ~60–75 minutes
**Prerequisites:** An AWS account with permissions to create EKS clusters, an EC2 instance (Amazon Linux 2023) to run commands from

---

## What You'll Learn

- How to set up an EC2 instance with the right IAM permissions to manage EKS
- How to install `kubectl`, `eksctl`, and `helm`
- How to create a production-style EKS cluster using `eksctl`
- How Kubernetes CPU/memory resource units work (`m` millicores, `Mi` mebibytes)
- How to install and verify the Metrics Server
- How the Horizontal Pod Autoscaler (HPA) calculates replica counts
- How to trigger real-time autoscaling using synthetic load
- What happens when your cluster runs out of CPU capacity (sets up Lab 2/3)

---

## Architecture Used in This Lab

| Component | Value |
|---|---|
| Cluster name | `scaling-demo-cluster` |
| Kubernetes version | `1.30` |
| Region | `ap-south-1` (change if needed) |
| Node type | `t3.medium` |
| Node count | 2 (min 2, max 5) |
| Sample app | `php-apache` (official K8s HPA demo image) |

---

## Step 0 — Attach an IAM Role to Your EC2 Instance

**Do this from the AWS Console (browser), not the EC2 terminal.**

1. Go to **IAM → Roles → Create role**.
2. Trusted entity type: `AWS service` → Use case: `EC2` → **Next**.
3. Attach permission policy: `AdministratorAccess` (fine for a lab/demo cluster).
4. Role name: `eksctl-admin-role` → **Create role**.
5. Go to **EC2 → Instances**, select your instance → **Actions → Security → Modify IAM role** → select `eksctl-admin-role` → **Update IAM role**.

### Verify from your EC2 terminal:

```bash
aws sts get-caller-identity
```

**Expected output** (your account ID and role ARN will differ):

```json
{
    "UserId": "AROAEXAMPLE:i-0abcd1234efgh5678",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/eksctl-admin-role/i-0abcd1234efgh5678"
}
```

>  If this fails with `Unable to locate credentials`, the role isn't attached yet, or you need to wait 10–15 seconds after attaching it.

---

## Step 1 — Install Required CLI Tools

### 1a. Check what's already installed

```bash
aws --version
kubectl version --client
eksctl version
helm version
```

Amazon Linux 2023 usually ships with AWS CLI v2 pre-installed. Install whatever is missing using the commands below.

### 1b. Install kubectl (v1.30, matched to our EKS version)

```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.14/2026-04-08/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin
cp ./kubectl $HOME/bin/kubectl
export PATH=$HOME/bin:$PATH
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
kubectl version --client
```

### 1c. Install eksctl

```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

### 1d. Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

### 1e. Confirm all tools

```bash
aws --version && kubectl version --client && eksctl version && helm version
```

All four should print clean version strings with no errors.

---

## Step 2 — Create the EKS Cluster

### 2a. Write the cluster config

```bash
cat <<'EOF' > cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: scaling-demo-cluster
  region: ap-south-1
  version: "1.30"

managedNodeGroups:
  - name: ng-scaling-demo
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 2
    maxSize: 5
    volumeSize: 20
    iam:
      withAddonPolicies:
        autoScaler: true
EOF
```

>  `withAddonPolicies.autoScaler: true` pre-attaches the IAM policy needed for the Cluster Autoscaler lab later, so we don't need to configure IRSA manually at that point.
>
> Change `region: ap-south-1` if you're deploying elsewhere — do this **before** creating the cluster, not after.

### 2b. Create the cluster

```bash
eksctl create cluster -f cluster-config.yaml
```

 **This takes 15–20 minutes.** eksctl provisions a VPC, subnets, NAT gateway, the EKS control plane, IAM roles, and the managed node group via CloudFormation.

>  **Tip:** Run this inside `tmux` in case your SSH session drops:
> ```bash
> tmux new -s ekscreate
> eksctl create cluster -f cluster-config.yaml
> # if disconnected: tmux attach -t ekscreate
> ```

### 2c. Verify cluster access

```bash
aws eks update-kubeconfig --region ap-south-1 --name scaling-demo-cluster
kubectl get nodes
```

**Expected output:**

```
NAME                                            STATUS   ROLES    AGE   VERSION
ip-192-168-x-x.ap-south-1.compute.internal      Ready    <none>   3m    v1.30.14-eks-xxxxxxx
ip-192-168-x-x.ap-south-1.compute.internal      Ready    <none>   3m    v1.30.14-eks-xxxxxxx
```

Both nodes must show `Ready` before continuing.

---

## Step 3 — Understand Your Node Capacity

Before deploying anything, let's see exactly how much CPU/memory is actually schedulable per node (this differs from the raw instance spec — Kubernetes reserves some for the kubelet and OS).

```bash
kubectl get nodes -o custom-columns='NAME:.metadata.name,CPU-CAP:.status.capacity.cpu,MEM-CAP:.status.capacity.memory,CPU-ALLOC:.status.allocatable.cpu,MEM-ALLOC:.status.allocatable.memory'
```

**Typical output for `t3.medium`:**

```
NAME                                            CPU-CAP   MEM-CAP     CPU-ALLOC   MEM-ALLOC
ip-192-168-x-x.ap-south-1.compute.internal      2         3931688Ki   1930m       3376680Ki
ip-192-168-x-x.ap-south-1.compute.internal      2         3931696Ki   1930m       3376688Ki
```

### Kubernetes resource units, quickly:

| Unit | Meaning |
|---|---|
| `1000m` (millicores) | = 1 full vCPU core |
| `500m` | = 0.5 vCPU (half a core) |
| `200m` | = 0.2 vCPU |
| `Mi` (mebibyte) | = 2²⁰ bytes ≈ 1.048576 MB (binary unit, not decimal `M`) |

Also check the **max pods per node** (capped by AWS VPC CNI, not just CPU/memory):

```bash
kubectl describe nodes | grep -A 8 "Allocatable:"
```

Look for the `pods:` line — on `t3.medium` this is typically `17` per node, meaning **34 pods max across 2 nodes**, regardless of CPU headroom. Keep this number in mind for later.

---

## Step 4 — Verify the Metrics Server

Newer EKS clusters often auto-provision Metrics Server as a managed community add-on. Check first before installing:

```bash
kubectl get pods -n kube-system | grep metrics-server
```

**If you see it already running (2/2 or similar), skip to the verification command below.**

### If it's NOT present, install it:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Wait 30–60 seconds, then verify:

```bash
kubectl get deployment metrics-server -n kube-system
```

>  **If it crash-loops** with a TLS/x509 certificate error (`kubectl logs -n kube-system <pod-name>`), patch it:
> ```bash
> kubectl patch deployment metrics-server -n kube-system --type='json' \
>   -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
> ```

### Confirm metrics are actually flowing (this is the real test):

```bash
kubectl top nodes
kubectl top pods -A
```

Both commands must return real numbers, not an error like `Metrics API not available`. **Do not proceed until this works.**

---

## Step 5 — Deploy the Sample CPU-Bound Application

We'll use `php-apache` — the official image from the Kubernetes HPA documentation, purpose-built for CPU-load demos.

```bash
cat <<'EOF' > php-apache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  labels:
    app: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
            memory: 100Mi
          limits:
            cpu: 500m
            memory: 200Mi
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    app: php-apache
spec:
  ports:
  - port: 80
  selector:
    app: php-apache
EOF

kubectl apply -f php-apache.yaml
```

### Verify:

```bash
kubectl get deployment php-apache
kubectl get pods -l app=php-apache
kubectl get svc php-apache
```

>  **Why `requests.cpu: 200m` matters:** HPA calculates target CPU utilization **as a percentage of this request value**, not an absolute number. Without a CPU request set, HPA cannot compute utilization at all — the target column will show `<unknown>`.
>
> The gap between `requests.cpu: 200m` and `limits.cpu: 500m` means that under full load, this pod can hit up to **250% utilization relative to its request** — useful for producing a clear, fast scaling demo.

---

## Step 6 — Create the HPA Resource

We'll target **25% CPU utilization**, scaling between 1 and 10 pods.

```bash
cat <<'EOF' > php-apache-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 25
EOF

kubectl apply -f php-apache-hpa.yaml
```

### Verify:

```bash
kubectl get hpa php-apache-hpa
```

**Expected output** (wait ~30 seconds if you see `<unknown>` — this is normal for the first metrics scrape):

```
NAME             REFERENCE               TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
php-apache-hpa   Deployment/php-apache   0%/25%        1         10        1          40s
```

### How HPA calculates replica count (important concept):

```
desiredReplicas = ceil( currentReplicas × ( currentMetricValue / desiredMetricValue ) )
```

This is **proportional scaling**, not a simple "crossed threshold → add one pod" trigger. Example: if CPU utilization hits 100% against a 25% target, the ratio is 4.0, so HPA jumps straight to 4× the current replica count in one step (capped by `maxReplicas`).

The HPA controller re-evaluates every **15 seconds** by default.

---

## Step 7 — Generate Load and Watch It Scale (Live Demo)

Open **three terminals/SSH sessions** to your EC2 instance (or `tmux` split panes).

### Terminal 1 — watch the HPA decision-making:

```bash
kubectl get hpa php-apache-hpa --watch
```

### Terminal 2 — watch pods scale out:

```bash
kubectl get pods -l app=php-apache --watch
```

### Terminal 3 — generate load (5 parallel busybox loops for sustained pressure):

```bash
for i in 1 2 3 4 5; do
  kubectl run load-generator-$i --image=busybox:1.36 --restart=Never -- /bin/sh -c \
    "while true; do wget -q -O- http://php-apache.default.svc.cluster.local; done"
done
```

### What to expect:

- Within ~1 minute, Terminal 1's `TARGETS` should climb well past 25%.
- Terminal 2 should show new `php-apache` pods appearing — likely jumping straight to several replicas due to proportional scaling.
- Scaling continues every ~15–30 seconds until either `maxReplicas: 10` is hit, or the cluster runs out of schedulable CPU (see note below).

>  **Optional deep-dive — push toward the CPU ceiling:**
> Recall from Step 3 that total allocatable CPU across 2 `t3.medium` nodes is ~3860m, and each pod requests 200m — meaning the cluster can schedule roughly **~15–18 `php-apache` pods** before running out of CPU (well below the 34-pod ENI limit). To observe this:
> ```bash
> kubectl patch hpa php-apache-hpa -p '{"spec":{"maxReplicas":20}}'
> ```
> Let load run 10–15 minutes. You should see some pods stuck in `Pending` state with an event like `Insufficient cpu` once the ceiling is hit — this is the exact condition that **Cluster Autoscaler (Lab 3)** is designed to resolve automatically.

### Let the load run for **10–15 minutes** to observe full convergence, then stop it:

```bash
for i in 1 2 3 4 5; do kubectl delete pod load-generator-$i; done
```

>  **Scale-down is intentionally slow.** HPA has a default 5-minute stabilization window to prevent flapping — don't be alarmed if replica count stays elevated for several minutes after load stops. Check the reason with:
> ```bash
> kubectl describe hpa php-apache-hpa
> ```
> Look for the `ScaleDownStabilized` condition in the output.

---

## Step 8 — Cleanup (Prepare for Next Lab)

Run these **in order** to tear down Lab 1 resources cleanly, so the cluster is fresh and ready for Lab 2 (Vertical Pod Autoscaler).

### 8a. Remove any leftover load generators (safe even if already deleted):

```bash
for i in 1 2 3 4 5; do kubectl delete pod load-generator-$i --ignore-not-found; done
kubectl delete pod load-generator --ignore-not-found
```

### 8b. Remove the HPA:

```bash
kubectl delete hpa php-apache-hpa
```

### 8c. Remove the php-apache app:

```bash
kubectl delete -f php-apache.yaml
```

### 8d. Verify a clean slate:

```bash
kubectl get hpa
kubectl get pods -l app=php-apache
kubectl get deployment php-apache
```

All three should return `No resources found` (or a "not found" error for the deployment) in the `default` namespace.

>  **Keep the cluster itself running** (`scaling-demo-cluster`) — Labs 2 and 3 will reuse it. Do **not** run `eksctl delete cluster` yet.

### If you're done with all labs and want to fully tear down the cluster:

```bash
eksctl delete cluster --name scaling-demo-cluster --region ap-south-1
```

 This deletes the VPC, node group, and control plane. Only run this at the very end of the entire course.

---

## Troubleshooting Quick Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| `Unable to locate credentials` | IAM role not attached / not propagated yet | Re-check Step 0, wait 15s after attaching |
| `kubectl: command not found` | PATH not updated in current shell | `export PATH=$HOME/bin:$PATH` or open new shell |
| `HPA TARGETS: <unknown>` | Metrics Server not scraped yet, or not installed | Wait 30s; check Step 4 |
| Metrics Server `CrashLoopBackOff` | Kubelet TLS cert not trusted | Apply `--kubelet-insecure-tls` patch (Step 4) |
| Pods stuck `Pending` under load | Node CPU exhausted | Expected — see Cluster Autoscaler note in Step 7 |
| HPA not scaling down after load stops | Default 5-min stabilization window | Expected behavior, not a bug — wait it out |

---

*End of Lab 1. Proceed to Lab 2: Vertical Pod Autoscaler (VPA) once cleanup is verified.*
