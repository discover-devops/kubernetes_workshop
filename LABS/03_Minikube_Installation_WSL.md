
* Windows 11
* WSL2 (Ubuntu)
* VS Code
* Git Bash on Windows
* Ubuntu running inside WSL2

The architecture will look like this:

```text
+------------------------------------------------------+
| Windows 11                                           |
|                                                      |
|  VS Code                                             |
|     │                                                |
|     ├── Git Bash                                     |
|     │                                                |
|     └── WSL Ubuntu ----------------------------------+
|             │
|             ├── Docker Engine
|             │
|             ├── Minikube
|             │
|             └── kubectl
|
+------------------------------------------------------+
```

The recommended stack is

```
Ubuntu (WSL2)
      │
Docker Engine
      │
Minikube (Docker Driver)
      │
Single Node Kubernetes Cluster
```

---

# Step 1 Verify WSL

You already verified it.

```bash
wsl --status
```

Output

```
Default Distribution: Ubuntu
Default Version: 2
```

Good.

---

# Step 2 Verify Ubuntu

Inside Ubuntu

```bash
uname -a
```

You already have

```
Linux ... microsoft-standard-WSL2
```

Perfect.

---

# Step 3 Update Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
```

---

# Step 4 Install Required Packages

```bash
sudo apt install -y \
curl \
wget \
apt-transport-https \
ca-certificates \
gnupg \
lsb-release
```

---

# Step 5 Install Docker Engine

First remove older versions if present.

```bash
sudo apt remove docker docker-engine docker.io containerd runc
```

Ignore errors if nothing is installed.

---

### Add Docker Repository

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
| sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Now add repository

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

---

Update

```bash
sudo apt update
```

Install Docker

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

# Step 6 Start Docker

If you have **systemd enabled inside WSL**

```bash
sudo systemctl start docker
```

Enable

```bash
sudo systemctl enable docker
```

Check

```bash
sudo systemctl status docker
```

---

If systemd is NOT enabled

Check

```bash
systemctl
```

If it says

```
System has not been booted with systemd
```

tell me.

We'll use another method.

---

# Step 7 Add User to Docker Group

```bash
sudo usermod -aG docker $USER
```

Then

```bash
exit
```

Close Ubuntu.

Open it again.

Verify

```bash
docker ps
```

No sudo should be required.

---

# Step 8 Verify Docker

```bash
docker version
```

Example

```
Client:
 Version: ...

Server:
 Engine:
 Version: ...
```

Also

```bash
docker run hello-world
```

Should work.

---

# Step 9 Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

```bash
chmod +x kubectl
```

```bash
sudo mv kubectl /usr/local/bin/
```

Verify

```bash
kubectl version --client
```

---

# Step 10 Install Minikube

Download

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```

Install

```bash
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

Verify

```bash
minikube version
```

---

# Step 11 Start Minikube

Since Docker is our runtime

```bash
minikube start --driver=docker
```

Minikube will

* Pull Kubernetes images
* Create Docker container
* Install Kubernetes
* Configure kubectl

---

# Step 12 Verify Cluster

```bash
kubectl get nodes
```

Expected

```
NAME       STATUS   ROLES           AGE
minikube   Ready    control-plane
```

---

Check Pods

```bash
kubectl get pods -A
```

You should see

* CoreDNS
* kube-apiserver
* kube-controller-manager
* kube-scheduler
* etcd
* kube-proxy

---

# Step 13 Verify Docker Driver

```bash
minikube profile list
```

Should show

```
Driver: docker
```

---

# Step 14 Verify Docker Containers

```bash
docker ps
```

You should see something like

```
minikube
```

The Kubernetes node itself runs as a Docker container.

---

# Step 15 Test Kubernetes

Create deployment

```bash
kubectl create deployment nginx --image=nginx
```

Check

```bash
kubectl get deployments
```

```bash
kubectl get pods
```

Expose

```bash
kubectl expose deployment nginx --type=NodePort --port=80
```

Access service

```bash
minikube service nginx
```

---

# One Important Question Before Proceeding

Before starting with Docker installation, please run these commands in your Ubuntu terminal and share the outputs:

```bash
docker version
```

```bash
docker ps
```

```bash
systemctl --version
```

```bash
cat /etc/wsl.conf
```

These will tell us:

* Whether Docker is already installed.
* Whether the Docker daemon is running.
* Whether `systemd` is enabled in your WSL2 instance.

Based on those outputs, we can either skip some steps or adjust the setup accordingly. This avoids installing or configuring components unnecessarily.
