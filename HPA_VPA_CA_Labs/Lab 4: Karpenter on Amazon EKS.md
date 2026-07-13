# Lab 4: Karpenter on Amazon EKS

**Level:** Advanced
**Duration:** ~75–90 minutes
**Prerequisites:** Completed **Lab 1 (HPA)**, **Lab 2 (VPA)**, and **Lab 3 (Cluster Autoscaler)**, with `scaling-demo-cluster` still running and cleaned up (2 nodes, no leftover CA/demo resources)

---

## What You'll Learn

- Why Karpenter exists, and the one core architectural difference that explains everything else about it
- The extra AWS wiring Karpenter needs compared to Cluster Autoscaler (OIDC/IRSA, node IAM role, discovery tags on subnets/security groups — not an ASG)
- How to install Karpenter v1 (`NodePool` / `EC2NodeClass` API) via Helm
- How to watch Karpenter select an instance type live, and why that decision differs fundamentally from what CA can do
- How Karpenter's consolidation feature works, and how it differs from CA's scale-down
- Real operational pitfalls: shell variable persistence across sessions, an incomplete IAM policy, and stuck CRD finalizers during cleanup

---

## The One Idea to Hold Onto

**Cluster Autoscaler can only launch the one instance type pre-configured in your node group's ASG. Karpenter looks at what the pending pod actually needs, and launches whichever EC2 instance type genuinely fits best — chosen fresh, every time, from the full instance catalog.**

Everything else people say about Karpenter (faster, cheaper, better bin-packing) is a *consequence* of this one fact — it isn't bound to a pre-defined group the way CA is.

| | Cluster Autoscaler (Lab 3) | Karpenter (Lab 4) |
|---|---|---|
| Talks to | AWS Auto Scaling Group API | EC2 API directly |
| Instance type choice | Fixed by the ASG's launch template — no choice | Chosen dynamically per batch of pending pods |
| Intermediate object | None (ASG desired-count change) | `NodeClaim` — Karpenter's own record of "I've decided to launch this" |
| Typical provisioning time | 2–4 minutes | Under 2 minutes, often under 1 |
| Scale-down behavior | Removes idle nodes after a cooldown | Actively **consolidates** — can replace several nodes with fewer/cheaper ones, not just remove idle ones |

---

## Step 1 — Set Environment Variables (Persist Them!)

>  **Critical lesson learned the hard way in this lab:** `export` only lasts for the current shell session. If you open a new terminal/SSH tab, or a session times out, all these variables silently reset to empty — and commands using them **don't error clearly**, they just substitute blank strings, causing confusing failures much later (wrong Helm chart version pulled, invalid empty tags, etc.). **Persist these in `~/.bashrc` from the start**, not just `export` them for the current session.

```bash
cat <<'EOF' >> ~/.bashrc

# Karpenter Lab environment variables
export CLUSTER_NAME=scaling-demo-cluster
export AWS_REGION=ap-south-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export KARPENTER_NAMESPACE=kube-system
export KARPENTER_VERSION=1.1.0
export K8S_VERSION=1.30
EOF

source ~/.bashrc
```

### Verify (do this before every subsequent step block, especially after any break):

```bash
echo "CLUSTER_NAME=[$CLUSTER_NAME]"
echo "AWS_REGION=[$AWS_REGION]"
echo "AWS_ACCOUNT_ID=[$AWS_ACCOUNT_ID]"
echo "KARPENTER_VERSION=[$KARPENTER_VERSION]"
```

All four must show non-empty values in brackets. If any show `[]`, re-run the export block above before continuing — **do not proceed with empty variables**, as later steps will fail in confusing ways (invalid empty AWS tags, wrong chart versions silently pulled, etc.).

---

## Step 2 — Associate an OIDC Provider (Required for IRSA)

Karpenter's controller uses **IRSA (IAM Roles for Service Accounts)** to get AWS permissions — a more tightly-scoped approach than Cluster Autoscaler's "permissions on the whole node role" (Lab 3). IRSA requires an OIDC identity provider associated with the cluster:

```bash
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --region $AWS_REGION --approve
```

### Verify:

```bash
aws eks describe-cluster --name $CLUSTER_NAME --region $AWS_REGION \
  --query "cluster.identity.oidc.issuer" --output text
```

Expect a URL like `https://oidc.eks.ap-south-1.amazonaws.com/id/XXXXXXXXX`.

---

## Step 3 — Create the Karpenter Node IAM Role

This is the role **new EC2 instances Karpenter launches** will assume — a separate role from your managed node group's, since Karpenter manages its own node fleet independently.

```bash
cat <<EOF > karpenter-node-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --assume-role-policy-document file://karpenter-node-trust-policy.json

aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam create-instance-profile --instance-profile-name "KarpenterNodeInstanceProfile-${CLUSTER_NAME}"
aws iam add-role-to-instance-profile \
  --instance-profile-name "KarpenterNodeInstanceProfile-${CLUSTER_NAME}" \
  --role-name "KarpenterNodeRole-${CLUSTER_NAME}"
```

### Register this role with the cluster via an EKS Access Entry (modern replacement for the old `aws-auth` ConfigMap approach):

```bash
aws eks create-access-entry \
  --cluster-name $CLUSTER_NAME --region $AWS_REGION \
  --principal-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME} \
  --type EC2_LINUX
```

> 📝 **Note:** An `EC2_LINUX` type access entry automatically grants the `system:nodes` Kubernetes group needed for node bootstrapping. Do **not** also try `aws eks associate-access-policy` on this entry — that command is only valid for `STANDARD` type entries (used for human/IAM-role `kubectl` access), and will correctly fail with `InvalidParameterException` if attempted here. This is expected, not an error to fix.

---

## Step 4 — Create the Karpenter Controller IAM Role (IRSA)

This is the role the **Karpenter pod itself** assumes, granting it permission to call the EC2 API directly.

```bash
cat <<EOF > karpenter-controller-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowScopedEC2InstanceActions",
      "Effect": "Allow",
      "Action": [
        "ec2:RunInstances",
        "ec2:CreateFleet",
        "ec2:CreateLaunchTemplate",
        "ec2:DeleteLaunchTemplate",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowScopedInstanceTermination",
      "Effect": "Allow",
      "Action": "ec2:TerminateInstances",
      "Resource": "*"
    },
    {
      "Sid": "AllowInstanceProfileReadActions",
      "Effect": "Allow",
      "Action": ["iam:GetInstanceProfile", "iam:PassRole"],
      "Resource": "*"
    },
    {
      "Sid": "AllowScopedInstanceProfileCreation",
      "Effect": "Allow",
      "Action": [
        "iam:CreateInstanceProfile",
        "iam:TagInstanceProfile",
        "iam:AddRoleToInstanceProfile",
        "iam:RemoveRoleFromInstanceProfile",
        "iam:DeleteInstanceProfile"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowDescribeActions",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeImages", "ec2:DescribeInstances", "ec2:DescribeSecurityGroups",
        "ec2:DescribeSubnets", "ec2:DescribeInstanceTypes", "ec2:DescribeInstanceTypeOfferings",
        "ec2:DescribeAvailabilityZones", "ec2:DescribeLaunchTemplates", "ec2:DescribeSpotPriceHistory",
        "pricing:GetProducts", "ssm:GetParameter", "eks:DescribeCluster"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name "KarpenterControllerPolicy-${CLUSTER_NAME}" \
  --policy-document file://karpenter-controller-policy.json
```

>  **Note the `ec2:DeleteLaunchTemplate` permission included above.** This was missing in an earlier draft of this policy and caused a real (non-fatal) error during node consolidation: `failed to delete launch template ... UnauthorizedOperation`. The EC2 instance itself still terminates correctly without it — only the launch template artifact is orphaned — but it's a genuine gap worth including from the start.

### Create the IRSA-scoped role and matching Kubernetes ServiceAccount:

```bash
eksctl create iamserviceaccount \
  --cluster $CLUSTER_NAME --region $AWS_REGION \
  --namespace $KARPENTER_NAMESPACE --name karpenter \
  --attach-policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME} \
  --role-name "KarpenterControllerRole-${CLUSTER_NAME}" \
  --approve
```

This scopes the role specifically to the `kube-system/karpenter` service account via the OIDC trust relationship — meaning *only* the Karpenter pod can assume it, not any pod on any node (unlike CA's node-role approach in Lab 3).

---

## Step 5 — Tag Subnets and Security Groups for Discovery

Karpenter finds where it's allowed to launch instances via tags — the equivalent concept to CA's ASG auto-discovery tags in Lab 3, but applied to network resources instead of an ASG.

```bash
SUBNET_IDS=$(aws eks describe-cluster --name $CLUSTER_NAME --region $AWS_REGION \
  --query "cluster.resourcesVpcConfig.subnetIds" --output text)

SECURITY_GROUP_ID=$(aws eks describe-cluster --name $CLUSTER_NAME --region $AWS_REGION \
  --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)

for SUBNET in $SUBNET_IDS; do
  aws ec2 create-tags --region $AWS_REGION --resources $SUBNET \
    --tags Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}
done

aws ec2 create-tags --region $AWS_REGION --resources $SECURITY_GROUP_ID \
  --tags Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}
```

### Verify:

```bash
aws ec2 describe-tags --region $AWS_REGION \
  --filters "Name=key,Values=karpenter.sh/discovery" \
  --query "Tags[].{Resource:ResourceId,Value:Value}" --output table
```

Should list every subnet ID plus the security group ID, all tagged with your cluster name.

---

## Step 6 — Install Karpenter via Helm

```bash
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace "${KARPENTER_NAMESPACE}" \
  --create-namespace \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "serviceAccount.create=false" \
  --set "serviceAccount.name=karpenter" \
  --wait
```

Notes:
- **`serviceAccount.create=false`** — the ServiceAccount already exists from Step 4's `eksctl create iamserviceaccount`; don't let Helm create a conflicting duplicate.
- **We deliberately omit `settings.interruptionQueue`.** See the troubleshooting note below — setting this without an actual SQS queue existing causes a hard crash.

> **Do not set `settings.interruptionQueue` unless you've created a real SQS queue for it.** Setting it to any value Karpenter can't resolve (e.g., just the cluster name, with no matching queue) causes the controller to **panic on startup** with `AWS.SimpleQueueService.NonExistentQueue`, resulting in `CrashLoopBackOff`. This setting is only needed for handling Spot instance interruptions — since this lab uses on-demand only, it's safe to omit entirely. In production with Spot, you'd provision the SQS queue + EventBridge rule first, then set this.

### Verify the deployment:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=karpenter
```

Expected: 2 pods, `1/1 Running`, 0 restarts.

### Check logs for clean startup:

```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter --tail=30
```

Should show controllers starting normally, no panics, no `AccessDenied`.

---

## Step 7 — Create the EC2NodeClass

Tells Karpenter **how** to configure new nodes: AMI family, IAM role, and how to find subnets/security groups via the discovery tags from Step 5.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  role: "KarpenterNodeRole-${CLUSTER_NAME}"
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}"
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}"
  amiSelectorTerms:
    - alias: al2023@latest
EOF
```

### Verify:

```bash
kubectl get ec2nodeclass default
kubectl describe ec2nodeclass default
```

Look for `Ready: True`, with sub-conditions `AMIsReady`, `SubnetsReady`, `SecurityGroupsReady`, and `InstanceProfileReady` all `True`.

>  **If you get `Invalid value: "object": empty tag keys or values aren't supported`**, this means `${CLUSTER_NAME}` resolved to an empty string when the heredoc ran — go back to Step 1 and confirm your environment variables are actually set in *this* shell session before retrying.

---

## Step 8 — Create the NodePool

Tells Karpenter **which** instance types it's allowed to choose from, and its disruption/consolidation behavior.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["t3.medium", "t3.large", "t3.xlarge"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      expireAfter: 720h
  limits:
    cpu: "20"
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
EOF
```

Key details:
- **`instance-type: In ["t3.medium", "t3.large", "t3.xlarge"]`** — the crucial difference from CA's single-instance-type ASG. Karpenter can now genuinely choose between three sizes based on what's actually needed.
- **`limits.cpu: "20"`** — a safety ceiling, similar in spirit to CA's `maxSize`.
- **`consolidateAfter: 1m`** — aggressive for demo purposes; production typically uses 5m+ to avoid unnecessary churn on bursty workloads.

### Verify:

```bash
kubectl get nodepool default
kubectl describe nodepool default
```

Look for `NodeClassReady: True`, `ValidationSucceeded: True`, `Ready: True`.

---

## Step 9 — Deploy a Workload and Watch Karpenter Choose an Instance Type

To genuinely demonstrate Karpenter's flexibility (not just "add another of the same"), deploy a workload sized so it **cannot** fit efficiently on `t3.medium` alone:

```bash
cat <<'EOF' > karpenter-demo-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: karpenter-demo-app
  labels:
    app: karpenter-demo-app
spec:
  replicas: 8
  selector:
    matchLabels:
      app: karpenter-demo-app
  template:
    metadata:
      labels:
        app: karpenter-demo-app
    spec:
      containers:
      - name: karpenter-demo-app
        image: registry.k8s.io/pause:3.9
        resources:
          requests:
            cpu: 900m
            memory: 200Mi
EOF

kubectl apply -f karpenter-demo-app.yaml
```

**Math:** 8 replicas × 900m = 7200m total needed. A `t3.medium` (~1930m allocatable) fits barely 2 pods. A `t3.xlarge` (~3920m allocatable) fits about 4 pods — meaning Karpenter needs **2× `t3.xlarge`** to fit all 8, not several small `t3.medium`s.

### Watch Karpenter's decision live:

```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter -f --tail=20
```

Look for `found provisionable pod(s)` → `computed new nodeclaim(s) to fit pod(s)` → `created nodeclaim` (showing the chosen `instance-types`) → `launched nodeclaim` → `registered nodeclaim` → `initialized nodeclaim`. This whole sequence typically completes in **under 2 minutes**.

### Watch the NodeClaims (Karpenter's own "I've decided to launch this" object — no CA equivalent):

```bash
kubectl get nodeclaims --watch
```

### Check the final result:

```bash
kubectl get nodes -L node.kubernetes.io/instance-type,karpenter.sh/nodepool
kubectl get pods -l app=karpenter-demo-app
```

**Expected:** your original 2× `t3.medium` (no `nodepool` label — untouched, belongs to the managed node group), plus 2 new nodes labeled `instance-type=t3.xlarge` and `nodepool=default`, with all 8 demo pods `Running`.

### Direct comparison with Lab 3:

| | CA (Lab 3): 6 pods needing 2400m, locked to `t3.medium` | Karpenter (Lab 4): 8 pods needing 7200m, free to choose |
|---|---|---|
| Nodes launched | 2× `t3.medium` | 2× `t3.xlarge` |
| Reasoning | Only one instance type available — no choice to make | Calculated real capacity need, picked the size that fits efficiently |
| Time to schedule | ~2-4 minutes | Under 2 minutes |

If this same 8-pod/900m workload had hit our Lab 3 CA setup (locked to `t3.medium` only), CA would have needed roughly `7200m ÷ 1930m` ≈ **4 `t3.medium` nodes** to fit everything — Karpenter reached the same total capacity with **2 larger, more efficient nodes** instead.

---

## Step 10 — Watch Consolidation Work

```bash
kubectl delete -f karpenter-demo-app.yaml
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter -f --tail=20
```

With `consolidateAfter: 1m` configured, look for a `disrupting nodeclaim(s)` event within about a minute — dramatically faster than CA's default 10-minute `scale-down-unneeded-time` from Lab 3.

```bash
watch -n 10 kubectl get nodes
```

Should settle back to your original 2 nodes.

---

## Step 11 — Cleanup

>  **Two critical lessons before you run this:**
> 1. **Confirm nodes have already scaled back down (via Step 10) before deleting the NodePool/EC2NodeClass or uninstalling Karpenter.** Deleting these objects while Karpenter is mid-consolidation, or uninstalling the controller before its cleanup finishes, can leave orphaned `NodeClaim`/`Node` objects stuck referencing already-terminated (or worse, still-running) EC2 instances.
> 2. **`EC2NodeClass` carries a finalizer** (`karpenter.k8s.aws/termination`). If you delete it after the controller is already gone, or mid-shutdown, the object can get stuck with a `deletionTimestamp` set but never actually removed — waiting forever on a finalizer that will never fire, since its owning controller is gone.

### 11a. Confirm workload cleanup already done (Step 10):

```bash
kubectl get pods -l app=karpenter-demo-app
kubectl get nodeclaims
kubectl get nodes
```

Should show no demo pods, no nodeclaims, and exactly 2 nodes before proceeding.

### 11b. Remove NodePool and EC2NodeClass:

```bash
kubectl delete nodepool default
kubectl delete ec2nodeclass default
```

### 11c. If EC2NodeClass gets stuck (shows up in `kubectl get ec2nodeclass` with a `deletionTimestamp` but won't fully delete):

First confirm no real EC2 instance is affected — check the AWS side directly if any node was `NotReady`:

```bash
aws ec2 describe-instances --region $AWS_REGION \
  --filters "Name=private-dns-name,Values=<the-stuck-node-name>" \
  --query "Reservations[].Instances[].{ID:InstanceId,State:State.Name}" --output table
```

If this returns empty (instance already terminated), it's safe to force-clear the stuck finalizer:

```bash
kubectl patch ec2nodeclass default --type='json' \
  -p='[{"op": "remove", "path": "/metadata/finalizers"}]'
```

### 11d. Uninstall Karpenter:

```bash
helm uninstall karpenter -n kube-system
```

### 11e. Final verification:

```bash
kubectl get nodes
kubectl get nodeclaims
kubectl get ec2nodeclass 2>&1
kubectl get nodepool 2>&1
kubectl get pods -n kube-system -l app.kubernetes.io/name=karpenter
```

Expected: exactly 2 nodes (`t3.medium` × 2), `No resources found` for nodeclaims/ec2nodeclass/nodepool, and no karpenter pods.

> 📝 **Optional — IAM cleanup:** the IAM roles, policies, and instance profile created in Steps 3–4 (`KarpenterNodeRole-*`, `KarpenterControllerRole-*`, `KarpenterControllerPolicy-*`, `KarpenterNodeInstanceProfile-*`) are not automatically removed and will persist in your AWS account. Clean these up manually via the IAM console or CLI if you want a fully pristine account state, though leaving them costs nothing while unused.

---

## This Concludes the Full Autoscaling Series

You've now covered all four layers, live, end-to-end:
1. **HPA** — scaling pod count based on CPU utilization
2. **VPA** — right-sizing pod CPU/memory requests automatically
3. **Cluster Autoscaler** — scaling node count within a fixed, pre-defined instance type
4. **Karpenter** — scaling node count *and* dynamically selecting the best-fitting instance type per workload

### If you're fully done with the course, tear down the entire cluster:

```bash
eksctl delete cluster --name scaling-demo-cluster --region ap-south-1
```

 This deletes the VPC, node group, control plane, and all associated CloudFormation stacks — **irreversible**. Only run this once you're certain no further labs will reuse this cluster.

### After cluster deletion, optionally clean up the Karpenter-specific IAM resources that eksctl won't remove automatically:

```bash
aws iam remove-role-from-instance-profile \
  --instance-profile-name "KarpenterNodeInstanceProfile-scaling-demo-cluster" \
  --role-name "KarpenterNodeRole-scaling-demo-cluster"
aws iam delete-instance-profile --instance-profile-name "KarpenterNodeInstanceProfile-scaling-demo-cluster"

aws iam detach-role-policy --role-name "KarpenterNodeRole-scaling-demo-cluster" --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam detach-role-policy --role-name "KarpenterNodeRole-scaling-demo-cluster" --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
aws iam detach-role-policy --role-name "KarpenterNodeRole-scaling-demo-cluster" --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
aws iam detach-role-policy --role-name "KarpenterNodeRole-scaling-demo-cluster" --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name "KarpenterNodeRole-scaling-demo-cluster"

# The KarpenterControllerRole-* is managed by eksctl's CloudFormation stack — clean it up via:
eksctl delete iamserviceaccount --cluster scaling-demo-cluster --region ap-south-1 \
  --namespace kube-system --name karpenter

aws iam delete-policy --policy-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/KarpenterControllerPolicy-scaling-demo-cluster
```

---

## Troubleshooting Quick Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| Any command using `$CLUSTER_NAME` etc. fails oddly (empty tags, wrong Helm version pulled) | Environment variables not set in current shell session | `echo` each variable to check for `[]`; persist via `~/.bashrc`, not just `export` |
| `helm install` pulls unexpected chart version (e.g. latest instead of pinned) | `--version` resolved to empty string due to unset variable | Re-check variables before every `helm` command in a long session |
| Karpenter pods `CrashLoopBackOff` with `panic: ... NonExistentQueue` | `settings.interruptionQueue` set to a non-existent SQS queue | Omit this setting entirely unless you've created a real SQS queue for Spot interruption handling |
| `EC2NodeClass` apply fails: `empty tag keys or values aren't supported` | `${CLUSTER_NAME}` empty when the heredoc ran | Confirm variables are set in current session before applying |
| Log shows `failed to delete launch template ... UnauthorizedOperation` | Controller IAM policy missing `ec2:DeleteLaunchTemplate` | Add this action to the controller policy (included in Step 4 above) |
| `EC2NodeClass`/`NodePool` deleted but still appears with a `deletionTimestamp` | Stuck finalizer — deleted while Karpenter was mid-consolidation or already uninstalled | Confirm underlying EC2 instance is genuinely terminated, then force-remove the finalizer via `kubectl patch` |
| Orphaned nodes/NodeClaims after cleanup | NodePool/EC2NodeClass deleted before consolidation finished | Always confirm `kubectl get nodeclaims` and `kubectl get nodes` show expected state *before* deleting NodePool/EC2NodeClass or uninstalling the controller |
| `aws eks associate-access-policy` fails: `InvalidParameterException` on an `EC2_LINUX` entry | This command only applies to `STANDARD` type access entries | Expected — `EC2_LINUX` entries auto-grant `system:nodes`; no further action needed |

---

*End of Lab 4 — and the complete four-part Kubernetes autoscaling series: HPA → VPA → Cluster Autoscaler → Karpenter, all demonstrated live on Amazon EKS.*
