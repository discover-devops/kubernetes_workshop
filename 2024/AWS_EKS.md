

### Setup Kubernetes on Amazon EKS

#### Prerequisites:

1. **EC2 Instance:**
   - Launch an EC2 instance.

2. **Install AWS CLI (latest version):**
   - Follow the installation instructions for AWS CLI.

3. **Setup kubectl:**
   a. Download kubectl version 1.21
   ```bash
   curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl
   chmod +x ./kubectl
   mv ./kubectl /usr/local/bin
   ```
   b. Test kubectl installation:
   ```bash
   kubectl version --short --client
   ```

4. **Setup eksctl:**
   a. Download and extract the latest release
   ```bash
   curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
   sudo mv /tmp/eksctl /usr/local/bin
   ```
   b. Test eksctl installation:
   ```bash
   eksctl version
   ```

5. **Create IAM Role:**
   - Create an IAM role and attach it to the EC2 instance.
   - The IAM user should have programmatic access and permissions for IAM, EC2, and CloudFormation.
   - Check eksctl documentation for minimum IAM policies.

#### Create Cluster and Nodes:

```bash
eksctl create cluster --name cluster-name \
--region region-name \
--node-type instance-type \
--nodes-min 2 \
--nodes-max 2 \
--zones <AZ-1>,<AZ-2>
```
Example:
```bash
eksctl create cluster --name my-cluster \
   --region ap-south-1 \
   --node-type t2.small
```

#### Delete EKS Cluster:

```bash
eksctl delete cluster my-cluster --region ap-south-1
```

#### Validate Cluster:

```bash
kubectl get nodes
kubectl run tomcat --image=tomcat
```

#### Deploy Nginx Pods:

```bash
kubectl create deployment demo-nginx --image=nginx --replicas=2 --port=80
kubectl get all
kubectl get pod
```

#### Expose Deployment as Service:

```bash
kubectl expose deployment demo-nginx --port=80 --type=LoadBalancer
kubectl get services -o wide
```


