# Lab 3: Cluster Autoscaler (CA) on Amazon EKS

**Level:** Intermediate–Advanced
**Duration:** ~60–75 minutes
**Prerequisites:** Completed **Lab 1 (HPA)** and **Lab 2 (VPA)**, with `scaling-demo-cluster` still running and cleaned up (2 nodes, no leftover HPA/VPA/demo resources)

---

## What You'll Learn

- How HPA, VPA, and Cluster Autoscaler fit together as three layers of autoscaling
- What an EKS Managed Node Group actually is under the hood (an AWS Auto Scaling Group)
- Why Cluster Autoscaler is open source and cloud-agnostic — and how it differs from Karpenter
- How to configure IAM permissions and ASG auto-discovery tags for Cluster Autoscaler
- How to install Cluster Autoscaler via Helm, matched to your cluster's Kubernetes version
- How to read Cluster Autoscaler's decision-making logs
- How to trigger a real scale-up by exhausting CPU capacity, and watch a real EC2 instance launch
- How scale-down cooldown works, and why it's intentionally slow
- Critical operational pitfalls: never manually touch a managed node group's ASG directly, and never tear down Cluster Autoscaler mid-scale-down

---

## Where This Fits: The Three Layers of Autoscaling

| | Scales | Answers the question |
|---|---|---|
| **HPA** (Lab 1) | Number of pods | "Do I need more copies of this app?" |
| **VPA** (Lab 2) | CPU/memory per pod | "Is each copy sized correctly?" |
| **Cluster Autoscaler** (Lab 3) | Number of nodes | "Do I have enough machines to run all these pods?" |

HPA and VPA operate entirely *within* the existing cluster capacity. Cluster Autoscaler is the layer that changes the capacity itself — it's the only one of the three that talks to AWS infrastructure APIs rather than just the Kubernetes API.

---

## Core Concept: What Is Cluster Autoscaler, Really?

**Cluster Autoscaler (CA) is a genuinely open source project** (`kubernetes/autoscaler` on GitHub), maintained by the Kubernetes community — not an AWS-specific or EKS-specific product. It supports a **pluggable cloud provider backend**: AWS, GCP, Azure, and others each have their own module inside the same codebase. We configure `--cloud-provider=aws`, which tells it to call the **AWS Auto Scaling API** specifically.

**It's one component talking to an external API — not two peer components.** The AWS Auto Scaling Group (ASG) isn't part of Cluster Autoscaler; it's pre-existing AWS infrastructure that CA calls into, the same way `kubectl` calls into the Kubernetes API server.

### The chain of custody behind an EKS "Managed Node Group"

```
EKS "Managed Node Group" (Kubernetes-facing concept)
        ↓ (is actually implemented as)
AWS Auto Scaling Group — an ASG (real EC2 infrastructure)
        ↓ (which launches/terminates)
EC2 instances → which register themselves as → Kubernetes Nodes
```

An EKS node group isn't a separate real thing — it's an abstraction on top of a plain ASG. The ASG is what actually tracks "I have 2 instances, min 2, max 5" and physically launches/terminates EC2 instances. Cluster Autoscaler runs as a pod *inside* Kubernetes but reaches *outside* to call the AWS Auto Scaling API (`SetDesiredCapacity`, `DescribeAutoScalingGroups`) to add or remove instances when pods can't be scheduled.

### Does CA require "managed" Kubernetes?

**No.** What it actually requires is: nodes that live inside *some* cloud provider's autoscaling primitive (an AWS ASG, a GCP Managed Instance Group, etc.). It works on EKS managed node groups (what we're doing), EKS self-managed node groups, and even self-built `kubeadm` clusters on EC2 — as long as those instances sit inside a correctly tagged ASG.

### A note on Karpenter (context, not required for this lab)

**Karpenter** is a newer, also-open-source, AWS-backed alternative that bypasses ASGs entirely — it provisions raw EC2 instances directly via the EC2 API, enabling faster, more flexible scaling decisions (e.g., right-sizing instance type per pending pod rather than "add another copy of whatever this ASG uses"). Many teams are migrating from Cluster Autoscaler to Karpenter on EKS. We're covering Cluster Autoscaler first because it's the more foundational, universally-applicable concept.

---

## Step 1 — Verify IAM Permissions Are Already in Place

Recall from Lab 1: our cluster config included `iam.withAddonPolicies.autoScaler: true` on the managed node group. This attaches the Cluster Autoscaler IAM policy **directly to the node instance role** — every pod running on these nodes inherits these permissions automatically.

>  **Production note:** this is a lab-appropriate shortcut. In production, you'd typically use **IRSA (IAM Roles for Service Accounts)** to scope these permissions to just the Cluster Autoscaler pod, not the entire node. This lab uses the simpler node-role approach since it was already configured back in Lab 1.

No action needed here — just be aware of *why* we can skip IRSA setup in this lab.

---

## Step 2 — Get Your Exact ASG Name

Don't rely on the truncated table output from `eksctl get nodegroup` for the ASG name — terminal width wrapping can silently corrupt it. Get it directly:

```bash
eksctl get nodegroup --cluster scaling-demo-cluster --region ap-south-1
```

Note the full `ASG NAME` column value, or extract it programmatically:

```bash
ASG_NAME=$(aws autoscaling describe-auto-scaling-groups \
  --region ap-south-1 \
  --query "AutoScalingGroups[?Tags[?Key=='eks:nodegroup-name' && Value=='ng-scaling-demo']].AutoScalingGroupName" \
  --output text)
echo "ASG Name: $ASG_NAME"
```

>  **Common pitfall — silent region mismatch:** if this (or any AWS CLI command in this lab) returns empty with no error, check your CLI's default region before assuming the resource doesn't exist:
> ```bash
> aws configure get region
> ```
> If blank or wrong, every command silently queries the wrong region and returns empty results — it looks like "nothing found" but is actually "looked in the wrong place." Fix with:
> ```bash
> aws configure set region ap-south-1
> ```

---

## Step 3 — Tag the ASG for Auto-Discovery

Cluster Autoscaler needs a safe, explicit way to know which ASGs it's allowed to manage — your AWS account may have many unrelated ASGs. These two tags solve that:

```bash
aws autoscaling create-or-update-tags --tags \
  ResourceId=${ASG_NAME},ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/enabled,Value=true,PropagateAtLaunch=true \
  ResourceId=${ASG_NAME},ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/scaling-demo-cluster,Value=owned,PropagateAtLaunch=true
```

| Tag | Meaning |
|---|---|
| `k8s.io/cluster-autoscaler/enabled = true` | "This ASG is fair game for *some* cluster-autoscaler to manage." |
| `k8s.io/cluster-autoscaler/scaling-demo-cluster = owned` | "Specifically, it belongs to the cluster named `scaling-demo-cluster`." |

>  **Note:** some documentation states eksctl auto-applies these tags to managed node groups. In practice, verify rather than assume:
> ```bash
> aws autoscaling describe-tags \
>   --filters "Name=auto-scaling-group,Values=${ASG_NAME}" \
>   --query "Tags[?contains(Key, 'k8s.io/cluster-autoscaler')]"
> ```
> If this returns `[]`, apply the tags manually as shown above — don't assume they're already there.

This is the *discoverability* half of the equation; `withAddonPolicies.autoScaler: true` (Step 1) was the *capability* half. Both are required together.

---

## Step 4 — Install Cluster Autoscaler via Helm

### 4a. Add the Helm repo

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update
```

### 4b. Find the chart version matching your cluster's Kubernetes version

CA's chart `APP VERSION` should align with your EKS version (`1.30` in our case) for correct compatibility:

```bash
helm search repo autoscaler/cluster-autoscaler --versions | grep "1.30\."
```

Note the `CHART VERSION` next to app version `1.30.x` (e.g., `9.37.0`).

### 4c. Install

```bash
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --version <CHART_VERSION_FROM_STEP_4b> \
  --set autoDiscovery.clusterName=scaling-demo-cluster \
  --set awsRegion=ap-south-1 \
  --set cloudProvider=aws \
  --set rbac.serviceAccount.create=true \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.skip-nodes-with-system-pods=false
```

Key flags explained:
- **`autoDiscovery.clusterName`** — must exactly match the ASG tag from Step 3, or CA finds zero manageable ASGs.
- **`cloudProvider=aws`** — selects the AWS backend module out of the multiple cloud providers this same open-source tool supports.
- **No IRSA annotation set** — since our node role already carries the IAM policy (Step 1), the CA pod inherits permissions through the node it runs on.
- **`skip-nodes-with-system-pods=false`** — by default CA won't scale down a node hosting non-DaemonSet system pods (e.g., CoreDNS), which can make scale-down behavior confusing on a small demo cluster. Setting this `false` allows more predictable scale-down for this lab; reconsider before using in production.

>  **Known issue — "Chart.yaml file is missing" on `helm install`:** if you hit this error even with a valid chart/version, it's typically a corrupted local Helm cache, not a broken chart. Fix:
> ```bash
> helm repo remove autoscaler
> rm -rf ~/.cache/helm ~/.config/helm ~/.local/share/helm
> helm repo add autoscaler https://kubernetes.github.io/autoscaler
> helm repo update
> ```
> If `helm install` *still* fails after this, pull the chart locally and install from the `.tgz` directly, bypassing repo-index resolution entirely:
> ```bash
> helm pull autoscaler/cluster-autoscaler --version <CHART_VERSION> --destination /tmp
> helm install cluster-autoscaler /tmp/cluster-autoscaler-<CHART_VERSION>.tgz \
>   --namespace kube-system \
>   --set autoDiscovery.clusterName=scaling-demo-cluster \
>   --set awsRegion=ap-south-1 \
>   --set cloudProvider=aws \
>   --set rbac.serviceAccount.create=true \
>   --set extraArgs.balance-similar-node-groups=true \
>   --set extraArgs.skip-nodes-with-system-pods=false
> ```

### 4d. Verify the deployment

The Helm chart's pod label uses `aws-cluster-autoscaler`, not just `cluster-autoscaler` — use the exact selector from the install NOTES output:

```bash
kubectl get pods -n kube-system -l "app.kubernetes.io/name=aws-cluster-autoscaler,app.kubernetes.io/instance=cluster-autoscaler"
```

Expected: `1/1 Running`.

### 4e. Check logs to confirm it's talking to AWS successfully

```bash
kubectl logs -n kube-system -l "app.kubernetes.io/name=aws-cluster-autoscaler,app.kubernetes.io/instance=cluster-autoscaler" --tail=50
```

Look for:
```
Found multiple availability zones for ASG "eks-ng-scaling-demo-..."
Starting main loop
No unschedulable pods
Skipping <node> - node group min size reached (current: 2, min: 2)
```

This confirms: CA found your tagged ASG, IAM permissions are working (no `AccessDenied`), and it correctly recognizes the cluster is already at minimum size with nothing to do.

---

## Step 5 — Deploy a Workload That Exceeds Current Capacity

Recall from Lab 1: 2× `t3.medium` nodes give ~1930m allocatable CPU each (~3860m total, shared with system pods). We'll deploy enough replicas at high enough CPU request to guarantee the scheduler runs out of room.

```bash
cat <<'EOF' > ca-demo-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ca-demo-app
  labels:
    app: ca-demo-app
spec:
  replicas: 15
  selector:
    matchLabels:
      app: ca-demo-app
  template:
    metadata:
      labels:
        app: ca-demo-app
    spec:
      containers:
      - name: ca-demo-app
        image: registry.k8s.io/pause:3.9
        resources:
          requests:
            cpu: 400m
            memory: 100Mi
          limits:
            cpu: 400m
            memory: 100Mi
EOF

kubectl apply -f ca-demo-app.yaml
```

- **15 replicas × 400m = 6000m total** — comfortably exceeds ~3860m total allocatable CPU, guaranteeing some pods can't be scheduled.
- **`registry.k8s.io/pause:3.9`** — the standard minimal "do-nothing" image for capacity tests. Pods don't need to do anything; they just need to *claim* CPU via their request.

### Confirm some pods are Pending due to insufficient CPU:

```bash
kubectl get pods -l app=ca-demo-app
kubectl describe pod -l app=ca-demo-app | grep -A 3 "Events:"
```

Look for: `0/2 nodes are available: 2 Insufficient cpu`.

---

## Step 6 — Watch Cluster Autoscaler React

```bash
kubectl logs -n kube-system -l "app.kubernetes.io/name=aws-cluster-autoscaler,app.kubernetes.io/instance=cluster-autoscaler" -f --tail=20
```

Within the next poll cycle (~10s intervals), look for a shift from `No unschedulable pods` to something like:

```
X unschedulable pods
Estimated X nodes needed
Scale-up: setting group eks-ng-scaling-demo-... size to N
Setting asg eks-ng-scaling-demo-... size to N
```

That last line is a **real AWS API call** (`SetDesiredCapacity`) triggered live from your cluster.

### In a second terminal, watch new nodes actually appear:

```bash
kubectl get nodes --watch
```

New EC2 instances take **2–4 minutes** to launch, boot, join the cluster, and pass health checks.

>  **Why you might see 2 new nodes instead of 1:** CA estimates upfront how many nodes are needed to fit *all* currently-unschedulable pods, then requests that many in a single batch — rather than trickling up one at a time. With 6 pending pods at `400m` each (`2400m` needed) and each `t3.medium` realistically fitting only ~3-4 such pods after per-node system pod overhead (kube-proxy, VPC CNI), CA may correctly conclude 1 additional node isn't enough and request 2 in one shot. Confirm all pods landed:
> ```bash
> kubectl get pods -l app=ca-demo-app
> ```
> All should show `Running`, and the CA log should show `No unschedulable pods` again — meaning everything was absorbed in a single scale-up pass.

### Understanding the scale-down eligibility check in the logs

Once nodes are running hot with your demo app, you'll see lines like:

```
Node ... unremovable: cpu requested (95.85% of allocatable) is above the scale-down utilization threshold
```

This is CA correctly refusing to consider scaling down any node that's still heavily utilized — removing it would just immediately re-trigger a scale-up.

---

## Step 7 — Trigger Scale-Down

```bash
kubectl delete -f ca-demo-app.yaml
```

```bash
kubectl top nodes
```

CPU utilization across all nodes should drop to near-idle almost immediately.

### Watch CA's logs for the scale-down decision:

```bash
kubectl logs -n kube-system -l "app.kubernetes.io/name=aws-cluster-autoscaler,app.kubernetes.io/instance=cluster-autoscaler" -f --tail=20
```

>  **Scale-down is intentionally much slower than scale-up, by design:**
> - A node must be **underutilized for a sustained period** — default `--scale-down-unneeded-time` is **10 minutes** — before CA even considers removing it. This prevents flapping (remove a node, immediately need it back).
> - Nodes are removed **one at a time**, with cooldown between removals, not all at once.
>
> Expect log lines like this to appear gradually over 10-15 minutes:
> ```
> ip-192-168-x-x... is unneeded since 2026-07-13 ...
> Scale-down: removing empty node ip-192-168-x-x...
> ```

### Instead of a scrolling `--watch`, poll periodically for a cleaner view:

```bash
watch -n 30 kubectl get nodes
```

Let this run 10–15 minutes. You should see node count drop back toward your ASG's `minSize` as CA removes now-unneeded nodes.

---

##  Critical Lessons Learned (Read Before Cleanup)

### Lesson 1: Never uninstall Cluster Autoscaler while scale-down is still pending

If you tear down CA (e.g., `helm uninstall`) before it has finished removing unneeded nodes, those extra nodes are **orphaned** — there's no controller left to notice they're idle and remove them. They'll sit there indefinitely, still costing money.

**If this happens, don't panic — but don't manually touch the ASG either (see Lesson 2). Fix it through the EKS-native path:**

```bash
eksctl scale nodegroup --cluster scaling-demo-cluster --name ng-scaling-demo \
  --nodes 2 --nodes-min 2 --nodes-max 5 --region ap-south-1
```

### Lesson 2: Never manually touch the raw ASG on an EKS Managed Node Group

It's tempting to fix a node-count problem directly via:

```bash
#  DON'T DO THIS on a managed node group
aws autoscaling update-auto-scaling-group --auto-scaling-group-name <ASG> --desired-capacity 2
```

**This can fight against EKS's own managed-node-group control loop**, which continuously reconciles the ASG back to what EKS itself tracks as the desired state. You may see confusing results — like the nodegroup still reporting the old desired capacity, or the *wrong* node being cordoned (`SchedulingDisabled`) instead of the intended one being removed.

**Always use the EKS-aware API instead:**

```bash
eksctl scale nodegroup --cluster scaling-demo-cluster --name ng-scaling-demo \
  --nodes 2 --nodes-min 2 --nodes-max 5 --region ap-south-1
```

This is the same principle as "don't edit generated files, edit the source" — the ASG is a generated artifact of the EKS node group, not the source of truth itself.

---

## Step 8 — Verify Final State Before Cleanup

```bash
kubectl get nodes
eksctl get nodegroup --cluster scaling-demo-cluster --region ap-south-1
```

Confirm you're back to exactly **2 nodes** and `DESIRED CAPACITY: 2` in the nodegroup output. If not, apply the `eksctl scale nodegroup` fix from Lesson 1/2 above before proceeding.

---

## Step 9 — Cleanup

### 9a. Remove the demo app (if not already deleted in Step 7):

```bash
kubectl delete -f ca-demo-app.yaml --ignore-not-found
```

### 9b. Uninstall Cluster Autoscaler — only after confirming node count is already settled (Step 8):

```bash
helm uninstall cluster-autoscaler -n kube-system
```

### 9c. Verify clean state:

```bash
kubectl get pods -l app=ca-demo-app
kubectl get pods -n kube-system | grep -E "cluster-autoscaler|vpa"
kubectl get nodes
```

Should show: no `ca-demo-app` resources, no `cluster-autoscaler` pods in `kube-system`, and exactly 2 nodes.

>  This concludes the three-lab autoscaling series. If you're fully done with the course, you can now tear down the cluster entirely:
> ```bash
> eksctl delete cluster --name scaling-demo-cluster --region ap-south-1
> ```
>  This deletes the VPC, node group, and control plane — only run this at the very end.

---

## Troubleshooting Quick Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| ASG queries return empty with no error | CLI default region mismatch | Check `aws configure get region`; set explicitly with `aws configure set region ap-south-1` |
| ASG has no `k8s.io/cluster-autoscaler/*` tags | Not auto-applied by eksctl in this setup | Tag manually (Step 3) — always verify, don't assume |
| `helm install` fails with "Chart.yaml file is missing" | Corrupted local Helm cache | Clear `~/.cache/helm`, `~/.config/helm`, `~/.local/share/helm` and retry; if still failing, `helm pull` + install from local `.tgz` |
| CA logs show `AccessDenied` / `UnauthorizedOperation` | IAM policy not actually attached to node role | Verify `withAddonPolicies.autoScaler: true` was set on the node group at creation |
| Pods stuck `Pending` with `Insufficient cpu`, no scale-up happening | CA not installed/running, or ASG tags missing, or ASG already at `maxSize` | Check CA pod is `Running`; verify tags (Step 3); check `eksctl get nodegroup` for current max |
| More nodes added than expected | CA batches scale-up to fit *all* pending pods at once | Expected behavior — verify with `kubectl get pods` that all landed in one pass |
| Nodes not scaling down after load removed | Default 10-minute `scale-down-unneeded-time` cooldown | Expected — wait it out; check CA logs for eligibility status |
| Orphaned extra nodes after uninstalling CA | CA removed before scale-down finished | Use `eksctl scale nodegroup --nodes <min>` to manually reconcile — never touch the ASG API directly |
| Wrong node cordoned / nodegroup desired count won't update | Manually modified ASG on a **managed** node group, fighting EKS's own control loop | Always use `eksctl scale nodegroup` (or the EKS API) instead of raw `aws autoscaling` commands on managed node groups |

---

*End of Lab 3 — and the full HPA → VPA → Cluster Autoscaler series. All three layers of Kubernetes autoscaling on EKS have now been demonstrated live, end-to-end.*
