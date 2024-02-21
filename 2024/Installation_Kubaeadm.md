
# Kubernetes Cluster Installation using kubeadm

Follow this documentation to set up a Kubernetes cluster on CentOS 7 machines. This guide will help you create a cluster with one master node and two worker nodes.

## Prerequisites

### System Requirements

- **Master:** t2.medium (2 CPUs and 2GB Memory)
- **Worker Nodes:** t2.micro

### Open Ports in Security Group

#### Master node:
- 6443
- 32750
- 10250
- 4443
- 443
- 8080

#### On Master and Worker:
- 179

### On Master and Worker:
- Perform all the commands as root user unless otherwise specified

## Installation Steps

### Install, Enable, and Start Docker Service

```bash
yum install -y -q yum-utils device-mapper-persistent-data lvm2 > /dev/null 2>&1
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo > /dev/null 2>&1
yum install -y -q docker-ce >/dev/null 2>&1
systemctl enable docker
systemctl start docker
```

### Disable SELinux

```bash
setenforce 0
sed -i --follow-symlinks 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux
```

### Disable Firewall

```bash
systemctl disable firewalld
systemctl stop firewalld
```

### Disable Swap

```bash
sed -i '/swap/d' /etc/fstab
swapoff -a
```

### Update sysctl settings for Kubernetes networking

```bash
cat >> /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

### Kubernetes Setup

#### Add yum repository for Kubernetes packages

```bash
cat >>/etc/yum.repos.d/kubernetes.repo<<EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

#### Install Kubernetes

```bash
yum install -y kubeadm-1.15.6-0.x86_64 kubelet-1.15.6-0.x86_64 kubectl-1.15.6-0.x86_64
systemctl enable kubelet
systemctl start kubelet
```

#### On Master Node:

##### Initialize Kubernetes Cluster

```bash
kubeadm init --apiserver-advertise-address=<MasterServerIP> --pod-network-cidr=192.168.0.0/16
```

##### Create a user for Kubernetes administration and copy kubeconfig file

```bash
useradd kubeadmin
mkdir /home/kubeadmin/.kube
cp /etc/kubernetes/admin.conf /home/kubeadmin/.kube/config
chown -R kubeadmin:kubeadmin /home/kubeadmin/.kube
```

##### Deploy Calico network as a kubeadmin user

```bash
sudo su - kubeadmin
kubectl create -f https://docs.projectcalico.org/v3.9/manifests/calico.yaml
```

##### Cluster join command

```bash
kubeadm token create --print-join-command
```

#### On Worker Node:

##### Add worker nodes to the cluster

Use the output from the `kubeadm token create` command from the master server and run it here.

### Verifying the Cluster

#### Get Nodes status

```bash
kubectl get nodes
```

#### Get component status

```bash
kubectl get cs
```

This completes the Kubernetes cluster installation and verification process.
```

Make sure to replace `<MasterServerIP>` with the actual IP address of your master node.
