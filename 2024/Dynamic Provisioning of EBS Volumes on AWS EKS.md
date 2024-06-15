### Dynamic Provisioning of EBS Volumes on AWS EKS

Dynamic provisioning of EBS volumes in Amazon EKS (Elastic Kubernetes Service) involves creating storage classes, persistent volume claims (PVCs), and using these claims in deployments. Here is a step-by-step guide along with the necessary YAML files.

### Prerequisites

1. **AWS EKS Cluster**: You need an existing EKS cluster.
2. **IAM Role with Required Permissions**: The worker nodes should have an IAM role with the necessary permissions to provision EBS volumes.

### Step-by-Step Guide

1. **Create a Storage Class**

A Storage Class defines how an EBS volume should be dynamically provisioned.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
  encrypted: "true"
```

2. **Create a Persistent Volume Claim (PVC)**

A PVC requests storage resources. Here’s how to create a PVC that uses the Storage Class defined above.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 20Gi
```

3. **Create a Deployment that Uses the PVC**

The deployment will use the PVC for storage.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: ebs-volume
      volumes:
      - name: ebs-volume
        persistentVolumeClaim:
          claimName: ebs-pvc
```

### Applying the YAML Files

To apply these YAML configurations, save each snippet to a file and then use the `kubectl apply` command:

```sh
kubectl apply -f storageclass.yaml
kubectl apply -f pvc.yaml
kubectl apply -f deployment.yaml
```

### Verifying the Setup

1. **Check Storage Class**

```sh
kubectl get storageclass
```

Ensure `ebs-sc` is listed.

2. **Check PVC**

```sh
kubectl get pvc
```

Ensure `ebs-pvc` is in the `Bound` state.

3. **Check Deployment**

```sh
kubectl get deployments
```

Ensure `nginx-deployment` is created and running.

4. **Check Pods**

```sh
kubectl get pods
```

Ensure the pods from the deployment are running.

### Detailed Steps Explanation

1. **Storage Class**:
   - The `StorageClass` named `ebs-sc` is configured to use the AWS EBS provisioner.
   - The `parameters` specify that the volume type is `gp2` (General Purpose SSD), the file system is `ext4`, and the volume should be encrypted.

2. **Persistent Volume Claim**:
   - The `PersistentVolumeClaim` named `ebs-pvc` requests 20Gi of storage using the `ebs-sc` StorageClass.
   - The `accessModes` specify that the volume can be mounted as read-write by a single node.

3. **Deployment**:
   - The `Deployment` named `nginx-deployment` creates two replicas of an NGINX container.
   - The `volumeMounts` section mounts the `ebs-volume` at `/usr/share/nginx/html`.
   - The `volumes` section specifies that the `ebs-volume` should use the `ebs-pvc` PersistentVolumeClaim.

By following this guide, you can dynamically provision EBS volumes for your deployments on AWS EKS, ensuring efficient and scalable storage management for your applications.
