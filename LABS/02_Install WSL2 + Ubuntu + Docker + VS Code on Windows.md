# Complete Step-by-Step Guide: Install WSL2 + Ubuntu + Docker + VS Code on Windows

This guide assumes:

* Windows 11 (or Windows 10 version 2004+)
* VS Code already installed
* Docker Desktop NOT required (we will install Docker Engine inside Ubuntu)
* Goal is to have a Linux development environment for Kubernetes, Docker, Git, Terraform, etc.

---

# Architecture

```
+------------------------------------------------------+
| Windows 11                                           |
|                                                      |
|  VS Code                                             |
|      │                                               |
|      ▼                                               |
|  Remote - WSL Extension                              |
|      │                                               |
|      ▼                                               |
| Ubuntu 24.04 (WSL2)                                  |
|      │                                               |
|      ▼                                               |
| Docker Engine                                        |
|      │                                               |
|      ▼                                               |
| Containers                                            |
+------------------------------------------------------+
```

---

# Step 1 Verify Windows Version

Open PowerShell

```powershell
winver
```

or

```powershell
systeminfo
```

You should have:

* Windows 11
* OR Windows 10 Version 2004+

---

# Step 2 Install WSL

Open **PowerShell as Administrator**

Run

```powershell
wsl --install
```

This command automatically installs:

* WSL2
* Virtual Machine Platform
* Ubuntu
* Linux Kernel

Wait until installation completes.

Restart Windows when prompted.

---

# Step 3 Verify WSL

After reboot

Open PowerShell

```powershell
wsl --status
```

Expected output

```
Default Version: 2

Kernel version:
...

Default Distribution:
Ubuntu
```

---

Check installed distributions

```powershell
wsl --list --verbose
```

Example

```
NAME      STATE      VERSION

Ubuntu    Running    2
```

Version must be **2**.

---

# Step 4 If Ubuntu isn't Installed

Install manually

List available distributions

```powershell
wsl --list --online
```

Example

```
Ubuntu
Ubuntu-24.04
Ubuntu-22.04
Debian
Kali
```

Install Ubuntu

```powershell
wsl --install Ubuntu-24.04
```

---

# Step 5 Launch Ubuntu

Run

```powershell
wsl
```

or

Search

```
Ubuntu
```

First launch takes a few minutes.

---

# Step 6 Create Linux User

Example

```
Enter new UNIX username:

rahul
```

Password

```
********
```

Done.

---

# Step 7 Update Ubuntu

Inside Ubuntu

```bash
sudo apt update
```

Then

```bash
sudo apt upgrade -y
```

---

# Step 8 Install Useful Packages

```bash
sudo apt install -y \
curl \
wget \
git \
vim \
nano \
zip \
unzip \
ca-certificates \
gnupg \
lsb-release
```

Verify Git

```bash
git --version
```

---

# Step 9 Verify WSL Filesystem

Windows Drive

```
/mnt/c
```

Example

```bash
cd /mnt/c
```

Your Windows C drive appears here.

Your Linux Home

```bash
cd ~
pwd
```

Example

```
/home/rahul
```

---

# Step 10 Install VS Code Extension

Open VS Code

Install

```
Remote - WSL
```

Extension by Microsoft.

---

# Step 11 Open Ubuntu inside VS Code

From Ubuntu terminal

```bash
code .
```

If the `code` command is not available yet, launch VS Code once from Windows and ensure the WSL extension is installed, then try again.

VS Code should automatically connect to WSL.

Bottom-left corner should show

```
WSL: Ubuntu
```

Now everything runs inside Linux.

---

# Step 12 Install Docker Engine

Update packages

```bash
sudo apt update
```

Install prerequisites

```bash
sudo apt install -y ca-certificates curl
```

Create key directory

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Download Docker GPG Key

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
-o /etc/apt/keyrings/docker.asc
```

Set permission

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker repository

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update

```bash
sudo apt update
```

Install Docker

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

# Step 13 Start Docker

If `systemd` is enabled in WSL:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

If `systemctl` isn't available, start the daemon manually in another terminal:

```bash
sudo dockerd
```

Keep that terminal running while using Docker.

---

# Step 14 Add Yourself to Docker Group

```bash
sudo usermod -aG docker $USER
```

Then exit WSL

```bash
exit
```

Reopen Ubuntu.

---

# Step 15 Verify Docker

```bash
docker version
```

Expected

```
Client:
...

Server:
...
```

---

Run

```bash
docker info
```

---

Run test container

```bash
docker run hello-world
```

Expected

```
Hello from Docker!
```

---

# Step 16 Verify Docker Compose

```bash
docker compose version
```

Example

```
Docker Compose version v2.xx.x
```

---

# Step 17 Create First Container

```bash
docker run -d \
--name nginx \
-p 8080:80 \
nginx
```

Verify

```bash
docker ps
```

Visit

```
http://localhost:8080
```

You should see the NGINX welcome page.

---

# Step 18 Useful Docker Commands

Images

```bash
docker images
```

Containers

```bash
docker ps
```

All containers

```bash
docker ps -a
```

Stop

```bash
docker stop nginx
```

Start

```bash
docker start nginx
```

Delete

```bash
docker rm -f nginx
```

---

# Step 19 Verify VS Code Integration

Open Ubuntu

```bash
mkdir dockerlab
```

```bash
cd dockerlab
```

```bash
code .
```

Create

```
Dockerfile
```

Create

```
docker-compose.yml
```

Everything is now inside Linux.

---

# Step 20 Install Kubernetes Tools (Optional)

Install `kubectl`

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/
```

Verify

```bash
kubectl version --client
```

---

Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

---

Install Kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64

chmod +x kind

sudo mv kind /usr/local/bin/
```

---

# Step 21 Install Terraform

```bash
sudo apt update

sudo apt install -y software-properties-common

wget -O- https://apt.releases.hashicorp.com/gpg \
| sudo gpg --dearmor \
-o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com \
$(lsb_release -cs) main" \
| sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update

sudo apt install terraform
```

---

# Step 22 Recommended VS Code Extensions

Install:

* WSL
* Docker
* Kubernetes
* YAML
* GitLens
* Terraform
* Dev Containers
* Remote Explorer

---

# Final Verification Checklist

Run these commands inside Ubuntu:

```bash
wsl --status          # Run from Windows PowerShell
```

```bash
docker version
```

```bash
docker compose version
```

```bash
git --version
```

```bash
kubectl version --client
```

```bash
terraform version
```

```bash
helm version
```

```bash
kind version
```

If all commands succeed, you'll have a complete Linux-based development environment on Windows suitable for Docker, Kubernetes, Terraform, Git, and cloud-native development.
