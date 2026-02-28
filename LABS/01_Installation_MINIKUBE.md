## Kubernetes Lab 1

## Installing Minikube on Ubuntu Using Docker Driver

### Overview

In this lab, we will set up a local Kubernetes cluster on an Ubuntu machine using Minikube and Docker. This setup is intended for learning and development purposes. By the end of this lab, you will have a working single-node Kubernetes cluster running locally.

You will:

* Install Docker
* Install Minikube
* Install kubectl
* Start a Kubernetes cluster
* Deploy and expose a sample application

---

## 1. Prerequisites

Before starting, ensure the following:

* Ubuntu 20.04 or 22.04 installed
* At least 2 CPU cores
* Minimum 2 GB RAM
* At least 20 GB free disk space
* User account with sudo privileges

You can verify system resources using:

Check CPU:

```
nproc
```

Check memory:

```
free -h
```

Check disk space:

```
df -h
```

---

## 2. Install Docker

Minikube will use Docker as the driver to create the Kubernetes node. So Docker must be installed first.

### Step 2.1: Update Package Index

```
sudo apt update
```

This refreshes the list of available packages from Ubuntu repositories.

---

### Step 2.2: Install Required Utilities

```
sudo apt install ca-certificates curl -y
```

* `ca-certificates` enables secure HTTPS communication.
* `curl` is used to download files from the internet.
* `-y` automatically confirms installation.

---

### Step 2.3: Create Docker Key Directory

```
sudo install -m 0755 -d /etc/apt/keyrings
```

This creates a secure directory where Docker’s repository keys will be stored.

---

### Step 2.4: Add Docker GPG Key

```
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
```

This downloads Docker’s official signing key so Ubuntu can verify the authenticity of Docker packages.

---

### Step 2.5: Add Docker Repository

```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list
```

This command adds Docker’s official repository to the system’s package sources.

---

### Step 2.6: Install Docker Engine

```
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y
```

This installs:

* Docker Engine
* Docker CLI
* Container runtime (containerd)

---

### Step 2.7: Allow Current User to Run Docker Without sudo

```
sudo usermod -aG docker $USER
```

This adds your user to the Docker group.

Apply the change:

```
newgrp docker
```

Verify Docker installation:

```
docker version
```

If it shows version details without using sudo, Docker is installed correctly.

---

## 3. Install Minikube

Minikube is used to create a local Kubernetes cluster.

### Step 3.1: Download Minikube

```
wget https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```

This downloads the latest Minikube binary.

---

### Step 3.2: Move Binary to System Path

```
sudo mv minikube-linux-amd64 /usr/local/bin/minikube
```

This moves the binary to a global location so it can be executed from anywhere.

---

### Step 3.3: Make It Executable

```
sudo chmod +x /usr/local/bin/minikube
```

This gives execution permission.

Verify installation:

```
minikube version
```

---

## 4. Install kubectl

kubectl is the command-line tool used to interact with Kubernetes.

### Step 4.1: Download kubectl

```
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
```

This downloads the latest stable version.

---

### Step 4.2: Make It Executable

```
chmod +x kubectl
```

---

### Step 4.3: Move to System Path

```
sudo mv kubectl /usr/local/bin/
```

Verify installation:

```
kubectl version --client
```

---

## 5. Start Minikube Cluster

Now we create the Kubernetes cluster.

```
minikube start --driver=docker
```

What this does:

* Creates a Kubernetes control plane inside a Docker container
* Configures networking
* Sets up kubectl context automatically

This process may take a few minutes.

---

## 6. Verify Cluster

Check nodes:

```
kubectl get nodes
```

You should see one node in Ready state.

Check system pods:

```
kubectl get pods -A
```

This shows all pods running in all namespaces.

---

## 7. Deploy a Sample Application

Create a deployment:

```
kubectl create deployment nginx --image=nginx
```

This creates a pod running the nginx container.

Expose the deployment as a service:

```
kubectl expose deployment nginx --type=NodePort --port=80
```

This makes the application accessible via a NodePort.

Access the application:

```
minikube service nginx
```

This will open the application in a browser.

---

## 8. Stop or Delete the Cluster

To stop the cluster:

```
minikube stop
```

To completely delete it:

```
minikube delete
```

Stopping preserves cluster state.
Deleting removes it entirely.

---

## Lab Outcome

After completing this lab, students should understand:

* The role of Docker in Kubernetes
* How Minikube creates a local cluster
* How kubectl interacts with Kubernetes
* How to deploy and expose an application
* How to verify cluster health

This completes Lab 1.
