Perfect Rahul 👍
I will create this exactly in your teaching style:

Context
Concept
Step by Step Lab
Command Purpose Explanation

This will be beginner friendly for students with limited Linux knowledge.

---

LAB 1
Install Minikube on Ubuntu Using Docker Driver
----------------------------------------------

CONTEXT

In this lab, we will create a local Kubernetes cluster on an Ubuntu machine.

We are not using cloud like Amazon EKS or Azure AKS.
We are creating a local cluster for development and learning.

Minikube will create:

Control Plane
Worker Node
Networking
Container Runtime

All inside your local machine.

Docker will be used as the container runtime.

---

WHAT WE ARE BUILDING

Ubuntu Machine
|
Docker Installed
|
Minikube Installed
|
kubectl Installed
|
Local Kubernetes Cluster Running

---

PREREQUISITES CHECK

Minimum Requirements

2 CPU cores
2 GB RAM
20 GB free disk
Ubuntu 20.04 or 22.04

Check CPU

Command
nproc

Purpose
Shows how many CPU cores are available.

Check memory

Command
free -h

Purpose
Shows available RAM.

Check disk

Command
df -h

Purpose
Shows disk usage.

---

STEP 1
Update Ubuntu Packages
----------------------

Command

sudo apt update

Purpose

sudo
Runs command as administrator.

apt
Ubuntu package manager.

update
Refresh package list from internet repositories.

This ensures system knows latest available packages.

---

STEP 2
Install Required Utilities
--------------------------

Command

sudo apt install ca-certificates curl -y

Purpose

ca-certificates
Allows secure HTTPS downloads.

curl
Used to download files from internet.

-y
Automatically answer yes to prompts.

---

STEP 3
Create Docker Keyring Directory
-------------------------------

Command

sudo install -m 0755 -d /etc/apt/keyrings

Purpose

install
Used to create directory.

-m 0755
Sets permissions.

-d
Creates directory.

This creates secure folder to store Docker repository keys.

---

STEP 4
Add Docker GPG Key
------------------

Command

sudo curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) -o /etc/apt/keyrings/docker.asc

Purpose

curl
Downloads file.

-f
Fail silently if error.

-s
Silent mode.

-S
Show errors.

-L
Follow redirects.

-o
Save output to file.

This downloads Docker security key so Ubuntu can trust Docker packages.

---

STEP 5
Add Docker Repository
---------------------

Command

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list

Purpose

echo
Prints text.

dpkg --print-architecture
Shows CPU architecture like amd64.

VERSION_CODENAME
Detects Ubuntu version.

tee
Writes output to file.

This tells Ubuntu where to download Docker from.

---

STEP 6
Install Docker
--------------

Command

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y

Purpose

docker-ce
Docker engine.

docker-ce-cli
Docker command line tool.

containerd.io
Container runtime backend.

Now Docker engine is installed.

---

STEP 7
Allow Current User to Run Docker Without sudo
---------------------------------------------

Command

sudo usermod -aG docker $USER

Purpose

usermod
Modifies user account.

-aG docker
Adds user to docker group.

$USER
Current logged in user.

This allows running docker commands without sudo.

Activate group change

Command

newgrp docker

Purpose

Refresh group membership without logging out.

Verify Docker

Command

docker version

If this works without sudo, Docker is ready.

---

STEP 8
Install Minikube
----------------

Download Minikube

Command

wget [https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64](https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64)

Purpose

wget
Downloads file from internet.

Move to system path

Command

sudo mv minikube-linux-amd64 /usr/local/bin/minikube

Purpose

/usr/local/bin
System wide command directory.

Make executable

Command

sudo chmod +x /usr/local/bin/minikube

Purpose

chmod +x
Makes file executable.

Verify

Command

minikube version

---

STEP 9
Install kubectl
---------------

Download kubectl

Command

curl -LO [https://storage.googleapis.com/kubernetes-release/release/$(curl](https://storage.googleapis.com/kubernetes-release/release/$%28curl) -s [https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl](https://storage.googleapis.com/kubernetes-release/release/stable.txt%29/bin/linux/amd64/kubectl)

Purpose

Downloads latest stable Kubernetes CLI tool.

Make executable

Command

chmod +x kubectl

Move to system path

Command

sudo mv kubectl /usr/local/bin/

Verify

Command

kubectl version --client

kubectl is now ready.

---

STEP 10
Start Minikube Cluster
----------------------

Command

minikube start --driver=docker

Purpose

minikube start
Creates Kubernetes cluster.

--driver=docker
Uses Docker to create Kubernetes node.

What happens internally

Docker container is created
Kubernetes control plane starts
Networking is configured
kubectl config is updated

---

STEP 11
Verify Cluster
--------------

Check nodes

Command

kubectl get nodes

Expected output

One node in Ready state.

Check pods

Command

kubectl get pods -A

Shows system pods running.

---

STEP 12
Deploy First Application
------------------------

Create deployment

Command

kubectl create deployment nginx --image=nginx

Purpose

Creates a pod running nginx container.

Expose service

Command

kubectl expose deployment nginx --type=NodePort --port=80

Purpose

Creates service so app can be accessed.

Get service URL

Command

minikube service nginx

This opens browser.

---

STEP 13
Stop and Delete Cluster
-----------------------

Stop cluster

minikube stop

Delete cluster

minikube delete

Purpose

Stop frees CPU and RAM.
Delete removes cluster completely.

---

LAB LEARNING OUTCOME

Students understand

What Docker does
What Minikube does
What kubectl does
How local Kubernetes works
Basic Linux command purpose
Cluster verification
Application deployment

---


