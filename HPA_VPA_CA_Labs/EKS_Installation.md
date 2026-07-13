
## Step 1 — IAM Role for the EC2 Instance (the right way to do this)

Don't use long-lived AWS access keys on an EC2 instance — attach an **IAM Role** to the instance itself. This is AWS's recommended approach and avoids storing credentials on disk.

**A. Create the IAM Role (do this once, via AWS Console or CLI from your local machine / another authenticated session):**

Go to **IAM → Roles → Create Role → AWS Service → EC2**, and attach these managed policies for this lab environment:
- `AdministratorAccess` (simplest for a learning/demo cluster — eksctl needs to create VPCs, subnets, IAM roles, CloudFormation stacks, EC2 instances, security groups, etc. Locking this down to least-privilege is a good follow-up exercise later, but it's a rabbit hole for day 1)

Name it something like `eksctl-admin-role`.

**B. Attach the role to your EC2 instance:**

Console → EC2 → select your instance → **Actions → Security → Modify IAM role** → attach `eksctl-admin-role`.

**C. Verify from your EC2 instance:**

```bash
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/
aws sts get-caller-identity
```

`get-caller-identity` should return your account ID and the role ARN. If this fails, the role isn't attached correctly — stop here and fix it before proceeding.

---

## Step 2 — Install AWS CLI v2 (Amazon Linux)

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo yum install -y unzip
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

Set your default region (adjust if you're not using Mumbai):
```bash
aws configure set region ap-south-1
aws configure set output json
```

---

## Step 3 — Install kubectl (version matched to EKS 1.30)

```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.14/2026-04-08/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl
export PATH=$HOME/bin:$PATH
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
kubectl version --client
```

---

## Step 4 — Install eksctl (official GitHub release)

```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

---

## Step 5 — Install Helm (needed later for VPA/Cluster Autoscaler labs)

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

---

## Step 6 — Create the EKS cluster with eksctl

This matches the target config you described: `scaling-demo-cluster`, 2× `t3.medium`, v1.30, with Cluster Autoscaler IAM policies pre-attached (so we don't have to configure IRSA manually in Lab 3).

```bash
cat <<'EOF' > cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: scaling-demo-cluster
  region: ap-south-1
  version: "1.30"

managedNodeGroups:
  - name: scaling-demo
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 2
    maxSize: 5
    volumeSize: 20
    iam:
      withAddonPolicies:
        autoScaler: true
EOF

eksctl create cluster -f cluster-config.yaml
```

This takes **15–20 minutes** — eksctl is provisioning a VPC, subnets, NAT gateway, IAM roles, the EKS control plane, and the managed node group via CloudFormation. Let it run; don't Ctrl+C.

---

## Step 7 — Verify cluster access

eksctl automatically updates `~/.kube/config`, but confirm explicitly:

```bash
aws eks update-kubeconfig --region ap-south-1 --name scaling-demo-cluster
kubectl get nodes
```

Expected:
```
NAME                                          STATUS   ROLES    AGE   VERSION
ip-192-168-x-x.ap-south-1.compute.internal    Ready    <none>   3m    v1.30.x-eks-xxxxx
ip-192-168-x-x.ap-south-1.compute.internal    Ready    <none>   3m    v1.30.x-eks-xxxxx
```
