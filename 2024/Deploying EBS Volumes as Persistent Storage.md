# Deploying EBS Volumes as Persistent Storage on Amazon EKS
### A Step-by-Step Tutorial — Kubernetes 1.31 / 1.32

---

## Table of Contents

1. [Core Concepts — The Mental Model](#1-core-concepts--the-mental-model)
   - 1.1 What is a CSI Driver?
   - 1.2 What is a StorageClass?
   - 1.3 What is a PersistentVolume (PV)?
   - 1.4 What is a PersistentVolumeClaim (PVC)?
   - 1.5 How They All Relate — The Full Picture
2. [Architecture Diagram](#2-architecture-diagram)
3. [Prerequisites](#3-prerequisites)
4. [Lab — Step-by-Step](#4-lab--step-by-step)
   - Step 1: Verify Your Cluster
   - Step 2: Create the IAM Role for the EBS CSI Driver
   - Step 3: Install the EBS CSI Driver Add-on
   - Step 4: Verify the Driver Installation
   - Step 5: Create a StorageClass
   - Step 6: Create a PersistentVolumeClaim
   - Step 7: Deploy a Pod That Uses the PVC
   - Step 8: Verify Storage Is Working
   - Step 9: Cleanup
5. [Troubleshooting Common Errors](#5-troubleshooting-common-errors)
6. [Key Takeaways](#6-key-takeaways)

---

## 1. Core Concepts — The Mental Model

Before running a single command, it's critical to understand what each piece does and **why** it exists. Think of the storage system in Kubernetes like renting office space:

---

### 1.1 What is a CSI Driver?

**CSI = Container Storage Interface.**

The CSI driver is the **translator and operator** between Kubernetes and the underlying cloud storage system (in this case, AWS EBS).

Kubernetes itself has no idea how to create an EBS volume, attach it to an EC2 node, or format it. The CSI driver bridges that gap. It runs as a set of pods inside your cluster and **watches for storage requests**, then calls the appropriate AWS APIs to fulfill them.

For EBS, the driver is: **`aws-ebs-csi-driver`**
- It runs as a **controller** (one pod, handles provisioning)
- And as a **node DaemonSet** (one pod per EC2 node, handles attach/mount)

> **Analogy:** The CSI driver is like a property manager. When a tenant (Pod) requests space, the manager (CSI) contacts the real estate company (AWS EBS API) to provision and hand over the keys.

---

### 1.2 What is a StorageClass?

A **StorageClass** is a **template / blueprint** that defines:
- *Which CSI driver* to use for provisioning
- *What type* of EBS volume to create (e.g., `gp3`, `gp2`, `io1`)
- *What the reclaim policy* is (delete or retain the volume when done)
- *Whether to wait* for a Pod to be scheduled before provisioning

You define a StorageClass once, and any number of PVCs can reference it.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com        # <-- tells Kubernetes: use the EBS CSI driver
parameters:
  type: gp3                          # <-- EBS volume type
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

> **Analogy:** The StorageClass is a menu item — "a gp3 EBS volume, 20GB, delete when done." You define the menu once; any Pod can order from it.

---

### 1.3 What is a PersistentVolume (PV)?

A **PersistentVolume** is the **actual storage resource** in the cluster — it represents a real EBS volume that exists in AWS.

PVs can be:
- **Statically provisioned** — You create an EBS volume manually in AWS and register it as a PV in Kubernetes.
- **Dynamically provisioned** — Kubernetes creates the EBS volume automatically when a PVC requests it (via a StorageClass). This is the modern, preferred approach.

A PV has:
- A **capacity** (e.g., 10Gi)
- An **access mode** (e.g., `ReadWriteOnce` — only one node can mount it at a time, which is an EBS limitation)
- A **reclaim policy** (what happens to the EBS volume when the PVC is released)

> **Analogy:** The PV is the actual physical office room. It has a size, a location, and rules about who can use it.

---

### 1.4 What is a PersistentVolumeClaim (PVC)?

A **PersistentVolumeClaim** is a **request for storage** made by a user or a Pod.

It says: *"I need 10Gi of storage with ReadWriteOnce access."*

Kubernetes looks at available PVs (or dynamically creates one via a StorageClass) and **binds** the PVC to a matching PV. Once bound, the PVC is referenced in a Pod spec, and the storage becomes available inside the container as a directory.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-ebs-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-gp3           # <-- references the StorageClass
  resources:
    requests:
      storage: 10Gi
```

> **Analogy:** The PVC is the tenant's rental application. "I need a 10 sqm office with private access." If a suitable room (PV) exists, it gets assigned. If not, the property manager (CSI) builds a new one.

---

### 1.5 How They All Relate — The Full Picture

```
Pod
 └── references ──► PVC  (the request)
                     └── bound to ──► PV  (the actual volume)
                                       └── backed by ──► EBS Volume in AWS
                                           provisioned by ──► CSI Driver
                                           configured by ──► StorageClass
```

**The request flow when using dynamic provisioning:**

1. You create a **StorageClass** (defines the "recipe")
2. A Pod references a **PVC** (makes a storage request)
3. Kubernetes sees the PVC and reads its `storageClassName`
4. The **EBS CSI Driver** (triggered by the StorageClass provisioner field) calls the AWS API to create an EBS volume
5. Kubernetes creates a **PV** object representing that EBS volume
6. The PVC is **bound** to the PV
7. The EBS CSI node DaemonSet **attaches** the volume to the EC2 node where the Pod is scheduled
8. The volume is **mounted** into the Pod's container at the specified path

---

## 2. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        EKS Cluster                          │
│                                                             │
│  ┌──────────┐    references    ┌──────────────────────────┐ │
│  │   Pod    │─────────────────►│  PersistentVolumeClaim   │ │
│  │          │                  │  (PVC)                   │ │
│  │ /data    │                  │  storage: 10Gi           │ │
│  └──────────┘                  │  class: ebs-gp3          │ │
│       ▲                        └────────────┬─────────────┘ │
│       │ mount                               │ binds to      │
│       │                                     ▼               │
│  ┌────┴─────────────────────────────────────────────────┐   │
│  │              PersistentVolume (PV)                   │   │
│  │              (auto-created by CSI driver)            │   │
│  └──────────────────────────┬───────────────────────────┘   │
│                             │ uses                          │
│  ┌──────────────────────────▼───────────────────────────┐   │
│  │              StorageClass: ebs-gp3                   │   │
│  │              provisioner: ebs.csi.aws.com            │   │
│  └──────────────────────────┬───────────────────────────┘   │
│                             │ calls                         │
│  ┌──────────────────────────▼───────────────────────────┐   │
│  │           EBS CSI Driver (kube-system)               │   │
│  │   Controller Pod + Node DaemonSet                    │   │
│  └──────────────────────────┬───────────────────────────┘   │
└───────────────────────────── │ ──────────────────────────────┘
                               │ AWS API calls
                               ▼
                    ┌─────────────────────┐
                    │   AWS EBS Volume    │
                    │   (gp3, 10GiB)      │
                    └─────────────────────┘
```

---

## 3. Prerequisites

Before starting the lab, ensure you have the following:

| Requirement | Check Command |
|---|---|
| EKS cluster running (1.31 or 1.32) | `kubectl version --short` |
| `kubectl` configured to your cluster | `kubectl get nodes` |
| `eksctl` installed | `eksctl version` |
| AWS CLI installed & configured | `aws sts get-caller-identity` |
| IAM permissions to create roles & policies | (verify with your admin) |

Also note the following EBS CSI limitations:
- EBS volumes support **`ReadWriteOnce`** only — they can be mounted by **one node at a time**
- EBS volumes are **AZ-specific** — a volume in `ap-south-1a` can only attach to a node in `ap-south-1a`
- EBS volumes **cannot** be used with Fargate Pods

---

## 4. Lab — Step-by-Step

### Step 1: Verify Your Cluster

First, confirm your cluster is healthy and note your cluster name and region. You'll need these throughout the lab.

```bash
# Confirm kubectl is connected to your cluster
kubectl get nodes

# Note your cluster name (you'll use this in later commands)
# Example output:
# NAME                                          STATUS   ROLES    AGE   VERSION
# ip-192-168-xx-xx.ap-south-1.compute.internal Ready    <none>   5d    v1.32.x
```

```bash
# Set shell variables for convenience (replace with your actual values)
export CLUSTER_NAME="my-eks-cluster"
export AWS_REGION="ap-south-1"    # Change to your region
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "Cluster: $CLUSTER_NAME | Region: $AWS_REGION | Account: $ACCOUNT_ID"
```

---

### Step 2: Create the IAM Role for the EBS CSI Driver

The EBS CSI driver needs IAM permissions to call AWS APIs (create volumes, attach them, etc.). We use **EKS Pod Identities** / **IRSA (IAM Roles for Service Accounts)** to grant these permissions securely.

#### 2a. Ensure OIDC Provider exists for your cluster

```bash
# Check if OIDC provider is already associated
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

If the output is a URL like `https://oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLE...`, you're good. If it's empty, create it:

```bash
# Create OIDC provider (only needed if not already present)
eksctl utils associate-iam-oidc-provider \
  --cluster $CLUSTER_NAME \
  --approve
```

#### 2b. Create the IAM Service Account with the EBS CSI Policy

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster $CLUSTER_NAME \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicyV2 \
  --approve
```

**What this does:**
- Creates an IAM Role named `AmazonEKS_EBS_CSI_DriverRole`
- Attaches the AWS-managed policy `AmazonEBSCSIDriverPolicyV2` (grants EC2 volume permissions)
- Configures the role to be assumable by the Kubernetes service account `ebs-csi-controller-sa` in the `kube-system` namespace

#### 2c. Verify the Role Was Created

```bash
aws iam get-role \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --query "Role.Arn" \
  --output text
```

Expected output: `arn:aws:iam::YOUR_ACCOUNT_ID:role/AmazonEKS_EBS_CSI_DriverRole`

---

### Step 3: Install the EBS CSI Driver Add-on

Install the driver as a managed EKS add-on. This is the recommended approach — AWS manages updates and compatibility.

```bash
# Get the Role ARN we just created
ROLE_ARN=$(aws iam get-role \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --query "Role.Arn" \
  --output text)

# Install the EBS CSI driver add-on
aws eks create-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn $ROLE_ARN \
  --region $AWS_REGION
```

Wait for it to become active (takes ~2 minutes):

```bash
# Watch the add-on status
aws eks describe-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name aws-ebs-csi-driver \
  --region $AWS_REGION \
  --query "addon.status"
```

Wait until you see: `"ACTIVE"`

---

### Step 4: Verify the Driver Installation

Confirm that the CSI driver pods are running:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

Expected output — you should see:
- **1 or 2** `ebs-csi-controller-*` pods (the controller, handles provisioning)
- **One `ebs-csi-node-*` pod per EC2 node** (the DaemonSet, handles attach/mount)

```
NAME                                  READY   STATUS    RESTARTS   AGE
ebs-csi-controller-7d4d74f5b4-abcde   6/6     Running   0          2m
ebs-csi-controller-7d4d74f5b4-fghij   6/6     Running   0          2m
ebs-csi-node-xxxxx                     3/3     Running   0          2m
ebs-csi-node-yyyyy                     3/3     Running   0          2m
```

Also verify the CSI driver is registered:

```bash
kubectl get csidriver ebs.csi.aws.com
```

Expected output:
```
NAME              ATTACHREQUIRED   PODINFOONMOUNT   STORAGECAPACITY   TOKENREQUESTS   REQUIRESREPUBLISH   MODES        AGE
ebs.csi.aws.com   true             false            false             <unset>         false               Persistent   2m
```

---

### Step 5: Create a StorageClass

Now define the StorageClass that will act as the blueprint for EBS volumes.

Create a file named `storageclass.yaml`:

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3                    # EBS volume type: gp3 (recommended), gp2, io1, io2
  encrypted: "true"            # Encrypt the EBS volume (best practice)
reclaimPolicy: Delete          # Delete the EBS volume when PVC is deleted
volumeBindingMode: WaitForFirstConsumer  # Wait until a Pod is scheduled before provisioning
                                         # This ensures the volume is in the correct AZ
allowVolumeExpansion: true     # Allow resizing the volume later
```

Apply it:

```bash
kubectl apply -f storageclass.yaml

# Verify
kubectl get storageclass ebs-gp3
```

Expected output:
```
NAME       PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ebs-gp3    ebs.csi.aws.com   Delete          WaitForFirstConsumer   true                   10s
```

> **Key parameter explained — `WaitForFirstConsumer`:** Without this, Kubernetes might provision an EBS volume in `ap-south-1a` but your Pod schedules on a node in `ap-south-1b`, causing a mount failure. This setting waits until the Pod's node is known, then creates the EBS volume in the correct AZ.

---

### Step 6: Create a PersistentVolumeClaim

Create a file named `pvc.yaml`:

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-demo-pvc
spec:
  accessModes:
    - ReadWriteOnce          # EBS supports ReadWriteOnce only (one node at a time)
  storageClassName: ebs-gp3  # Must match the StorageClass name from Step 5
  resources:
    requests:
      storage: 10Gi          # Size of the EBS volume to create
```

Apply it:

```bash
kubectl apply -f pvc.yaml

# Check the PVC status
kubectl get pvc ebs-demo-pvc
```

Expected output:

```
NAME           STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ebs-demo-pvc   Pending                                      ebs-gp3        5s
```

> **Why `Pending`?** Because `volumeBindingMode: WaitForFirstConsumer` means the volume won't be provisioned until a Pod references this PVC and gets scheduled to a node. This is expected behavior.

---

### Step 7: Deploy a Pod That Uses the PVC

Create a file named `pod.yaml`:

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ebs-demo-pod
spec:
  containers:
    - name: app
      image: nginx:stable
      ports:
        - containerPort: 80
      volumeMounts:
        - name: data-volume
          mountPath: /usr/share/nginx/html/data  # Path inside the container
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: ebs-demo-pvc                  # Must match the PVC name from Step 6
```

Apply it:

```bash
kubectl apply -f pod.yaml

# Watch the Pod come up
kubectl get pod ebs-demo-pod -w
```

While the Pod is starting, you'll see it go through these stages:
```
NAME           READY   STATUS              RESTARTS   AGE
ebs-demo-pod   0/1     Pending             0          5s
ebs-demo-pod   0/1     ContainerCreating   0          20s   # EBS volume being attached here
ebs-demo-pod   1/1     Running             0          40s
```

> **What's happening during `ContainerCreating`?** The EBS CSI driver is calling the AWS EC2 API to:
> 1. Create the EBS volume in the correct AZ
> 2. Attach it to the EC2 node where the Pod is scheduled
> 3. Format it with ext4 filesystem
> 4. Mount it into the container

---

### Step 8: Verify Storage Is Working

#### 8a. Check PVC and PV are now bound

```bash
kubectl get pvc ebs-demo-pvc
```

Expected output:
```
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ebs-demo-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   10Gi       RWO            ebs-gp3        2m
```

```bash
# Also see the auto-created PV
kubectl get pv
```

#### 8b. Verify the EBS Volume exists in AWS

```bash
# Get the volume ID from the PV
VOLUME_ID=$(kubectl get pv \
  $(kubectl get pvc ebs-demo-pvc -o jsonpath='{.spec.volumeName}') \
  -o jsonpath='{.spec.csi.volumeHandle}')

echo "EBS Volume ID: $VOLUME_ID"

# Confirm it exists in AWS
aws ec2 describe-volumes \
  --volume-ids $VOLUME_ID \
  --query "Volumes[0].{ID:VolumeId,State:State,Size:Size,AZ:AvailabilityZone,Type:VolumeType}" \
  --region $AWS_REGION
```

Expected output:
```json
{
    "ID": "vol-xxxxxxxxxxxxxxxxx",
    "State": "in-use",
    "Size": 10,
    "AZ": "ap-south-1a",
    "Type": "gp3"
}
```

#### 8c. Write and read data to confirm persistence

```bash
# Write a file to the mounted EBS volume
kubectl exec ebs-demo-pod -- \
  sh -c "echo 'Hello from EBS storage! $(date)' > /usr/share/nginx/html/data/test.txt"

# Read it back
kubectl exec ebs-demo-pod -- \
  cat /usr/share/nginx/html/data/test.txt
```

Expected output:
```
Hello from EBS storage! Sat May 09 12:00:00 UTC 2026
```

#### 8d. Test persistence across Pod restarts

```bash
# Delete the Pod
kubectl delete pod ebs-demo-pod

# Recreate it (using the same YAML)
kubectl apply -f pod.yaml

# Wait for it to be Running
kubectl get pod ebs-demo-pod -w

# Read the file again — data should still be there!
kubectl exec ebs-demo-pod -- \
  cat /usr/share/nginx/html/data/test.txt
```

If the data is still there, your EBS persistent storage is working correctly.

---

### Step 9: Cleanup

When you're done with the lab, clean up all resources to avoid AWS charges.

```bash
# Delete the Pod first
kubectl delete pod ebs-demo-pod

# Delete the PVC (this also deletes the underlying EBS volume, because reclaimPolicy: Delete)
kubectl delete pvc ebs-demo-pvc

# Verify the PV is gone (it will be Released then Deleted automatically)
kubectl get pv

# Delete the StorageClass
kubectl delete storageclass ebs-gp3
```

> **Important:** Because we set `reclaimPolicy: Delete` in the StorageClass, deleting the PVC will automatically delete the EBS volume in AWS. Verify in the console or with:
> ```bash
> aws ec2 describe-volumes --volume-ids $VOLUME_ID --region $AWS_REGION
> # Should return: "InvalidVolume.NotFound"
> ```

If you want to **remove the EBS CSI driver add-on** entirely:

```bash
aws eks delete-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name aws-ebs-csi-driver \
  --region $AWS_REGION
```

---

## 5. Troubleshooting Common Errors

| Error | Cause | Fix |
|---|---|---|
| `failed to provision volume: UnauthorizedOperation` | IAM Role missing or wrong | Re-check Step 2; verify the role ARN is correct in the add-on |
| PVC stays in `Pending` forever | No Pod referencing it, or AZ mismatch | Ensure a Pod uses the PVC; check `WaitForFirstConsumer` setting |
| Pod stuck in `ContainerCreating` | Volume attach timeout or node issue | `kubectl describe pod <name>` to see events; check CSI driver logs |
| `VolumeMount error: Multi-Attach` | Trying to mount EBS on two nodes simultaneously | EBS only supports `ReadWriteOnce`; use EFS for shared access |
| `no storage class found` | StorageClass name mismatch | Ensure `storageClassName` in PVC matches exactly |

Check CSI driver logs for detailed errors:

```bash
# Controller logs (provisioning issues)
kubectl logs -n kube-system -l app=ebs-csi-controller -c csi-provisioner --tail=50

# Node logs (attach/mount issues)
kubectl logs -n kube-system -l app=ebs-csi-node -c ebs-plugin --tail=50
```

---

## 6. Key Takeaways

| Concept | One-liner Summary |
|---|---|
| **CSI Driver** | The bridge between Kubernetes and AWS EBS APIs |
| **StorageClass** | A template/recipe for how to create EBS volumes |
| **PersistentVolume (PV)** | The actual EBS volume, represented as a Kubernetes object |
| **PersistentVolumeClaim (PVC)** | A Pod's request for storage; gets bound to a PV |
| **Dynamic Provisioning** | Kubernetes + CSI auto-creates the EBS volume when a PVC is submitted |
| **WaitForFirstConsumer** | Ensures the EBS volume is created in the same AZ as the scheduled Pod |
| **ReadWriteOnce** | EBS volumes can only be mounted by one node at a time |

---

*Tutorial based on AWS EKS documentation: https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html*
*Kubernetes version: 1.31 / 1.32 | Tool: eksctl*
