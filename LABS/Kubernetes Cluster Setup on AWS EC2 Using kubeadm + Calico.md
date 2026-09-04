# Kubernetes Cluster Setup on AWS EC2 Using kubeadm + Calico

## Objective

In this runbook, we will create a Kubernetes cluster using:

* **1 Control Plane / Master node**
* **2 Worker nodes**
* Ubuntu 24.04 LTS
* containerd as the container runtime
* kubeadm for cluster bootstrapping
* Calico as the CNI/network plugin

At the end, you should have:

<img width="1186" height="705" alt="image" src="https://github.com/user-attachments/assets/33c6e56b-1593-493a-adb4-2213c6d3f0c4" />


---

# Step 1 — Create the EC2 Instances

Create **3 EC2 instances**.

| Instance        | Role          | OS               |
| --------------- | ------------- | ---------------- |
| `k8s-master`    | Control Plane | Ubuntu 24.04 LTS |
| `k8s-worker-01` | Worker        | Ubuntu 24.04 LTS |
| `k8s-worker-02` | Worker        | Ubuntu 24.04 LTS |

Use the same:

* VPC
* Subnet
* Security Group

Make sure the instances can communicate with each other using their **private IP addresses**.

For example:

```text
Master       10.0.1.172
Worker-01    10.0.1.97
Worker-02    10.0.1.232
```

Your IP addresses will be different.

---

# Step 2 — Configure the Security Group

Allow SSH:

```text
TCP 22
```

Allow Kubernetes API:

```text
TCP 6443
```

Allow kubelet:

```text
TCP 10250
```

For our voting application later, allow:

```text
TCP 30001
TCP 30002
TCP 30003
```

For the Kubernetes nodes themselves, make sure the nodes can communicate with each other over the required Kubernetes networking ports.

**Recommended:** allow the Kubernetes node-to-node traffic within the VPC/security group rather than exposing these ports to the internet.

---

# Step 3 — Perform the Base Setup on ALL 3 Nodes

SSH into each machine.

Run the following on:

* `k8s-master`
* `k8s-worker-01`
* `k8s-worker-02`

## Update packages

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

Install basic utilities:

```bash
sudo apt-get install -y \
  ca-certificates \
  curl \
  gpg \
  apt-transport-https \
  software-properties-common
```

---

# Step 4 — Disable Swap

Kubernetes expects swap to be disabled for this setup.

Check:

```bash
free -h
```

Disable swap:

```bash
sudo swapoff -a
```

Disable it permanently:

```bash
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Verify:

```bash
swapon --show
```

It should return nothing.

---

# Step 5 — Configure Kernel Modules

This is important for Kubernetes networking.

Run on **all 3 nodes**:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

Load the modules:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Verify:

```bash
lsmod | grep br_netfilter
lsmod | grep overlay
```

---

# Step 6 — Configure iptables / Network Forwarding

This is one of the areas that can cause networking problems if it is missed.

Run on **all 3 nodes**:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply:

```bash
sudo sysctl --system
```

Verify:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

### Why do we need this?

Kubernetes Pods communicate across nodes, and the CNI needs Linux networking and packet forwarding to work correctly.

If these settings are incorrect, you can end up with:

* Pods unable to communicate
* Services not working
* DNS failures
* CNI problems

---

# Step 7 — Install containerd

We will use **containerd** as our container runtime.

Run on **all 3 nodes**:

```bash
sudo apt-get update
sudo apt-get install -y containerd
```

Create the configuration directory:

```bash
sudo mkdir -p /etc/containerd
```

Generate the default configuration:

```bash
containerd config default | sudo tee /etc/containerd/config.toml
```

Now configure containerd to use the systemd cgroup driver:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Verify:

```bash
grep SystemdCgroup /etc/containerd/config.toml
```

You **must** see:

```text
SystemdCgroup = true
```

Restart containerd:

```bash
sudo systemctl restart containerd
```

Enable it at boot:

```bash
sudo systemctl enable containerd
```

Check:

```bash
sudo systemctl status containerd --no-pager
```

You should see:

```text
Active: active (running)
```

### Important

Do not continue until containerd is running correctly on **all 3 nodes**.

---

# Step 8 — Install kubeadm, kubelet and kubectl

We need three Kubernetes components:

### kubeadm

Used to create and join the cluster.

### kubelet

Runs on every node and manages Kubernetes Pods.

### kubectl

Command-line tool used to communicate with the Kubernetes API server.

Install the Kubernetes packages on **all 3 nodes**.

Use the current Kubernetes package repository for the Kubernetes minor version you have selected. The repository must match across all three nodes.

After installation, verify:

```bash
kubeadm version
kubelet --version
kubectl version --client
```

Enable kubelet:

```bash
sudo systemctl enable kubelet
```

The kubelet may not be fully running yet because the cluster has not been initialized. That is normal.

---

# Step 9 — Initialize the Control Plane

Now switch to:

```text
K8s_Master
```

Find the master's **private IP**:

```bash
hostname -I
```

Suppose it is:

```text
10.0.1.172
```

Your IP will be different.

Initialize Kubernetes:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=10.0.1.172 \
  --pod-network-cidr=192.168.0.0/16
```

### Very important

Replace:

```text
10.0.1.172
```

with your master's actual **private IP**.

Do **not** remove:

```text
--pod-network-cidr=192.168.0.0/16
```

This is important for our Calico configuration.

---

# Step 10 — Configure kubectl on the Master

After `kubeadm init` successfully completes, run:

```bash
mkdir -p $HOME/.kube
```

```bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Test:

```bash
kubectl get nodes
```

Initially you may see:

```text
k8s-master   NotReady
```

That is expected.

Why?

Because **the Kubernetes control plane exists, but we haven't installed the Pod network yet.**

---

# Step 11 — Install Calico

We will use the stable **Calico v3.32.2** release.

Do this **only on the master**.

First install the Calico CRDs:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.2/manifests/v1_crd_projectcalico_org.yaml
```

Then install the Tigera Operator:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.2/manifests/tigera-operator.yaml
```

Then create the Calico resources:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.2/manifests/custom-resources.yaml
```

Watch the Pods:

```bash
kubectl get pods -A -w
```

You should eventually see Calico components such as:

```text
calico-apiserver
calico-kube-controllers
calico-node
calico-typha
```

becoming `Running` and `Ready`.

Check:

```bash
kubectl get pods -A
```

And:

```bash
kubectl get tigerastatus
```

---

# Step 12 — Verify Pod CIDR

This is a very useful troubleshooting check.

Run:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.podCIDR}{"\n"}{end}'
```

You should now see Pod CIDRs assigned.

For example:

```text
k8s-master       192.168.0.0/24
```

Workers will receive their own Pod CIDRs after they join.

If the Pod CIDRs are completely blank, **stop here and troubleshoot before continuing**.

---

# Step 13 — Generate the Worker Join Command

On the master:

```bash
kubeadm token create --print-join-command
```

You will get something similar to:

```bash
kubeadm join 10.0.1.172:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

**Do not copy the example above.**

Copy the actual command generated by your cluster.

---

# Step 14 — Join Worker 01

SSH into:

```text
k8s-worker-01
```

Run the generated command with `sudo`.

For example:

```bash
sudo kubeadm join 10.0.1.172:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

You should eventually see:

```text
This node has joined the cluster
```

---

# Step 15 — Join Worker 02

SSH into:

```text
k8s-worker-02
```

Run the **same generated join command**:

```bash
sudo kubeadm join 10.0.1.172:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

# Step 16 — Verify the Cluster

Return to the master.

Run:

```bash
kubectl get nodes
```

Expected:

```text
NAME             STATUS   ROLES           AGE
k8s-master       Ready    control-plane
k8s-worker-01    Ready    <none>
k8s-worker-02    Ready    <none>
```

Then:

```bash
kubectl get pods -A
```

The important system components should be healthy.

You should see:

```text
calico-*                    Running
coredns-*                   Running
kube-proxy-*                Running
etcd-*                      Running
kube-apiserver-*            Running
kube-controller-manager-*   Running
kube-scheduler-*            Running
tigera-operator-*           Running
```

---

# Step 17 — Verify Calico

Check:

```bash
kubectl get pods -n calico-system
```

You want the Calico Pods to show `Ready`.

For example:

```text
NAME                           READY   STATUS
calico-apiserver-xxxxx         1/1     Running
calico-kube-controllers-xxxxx  1/1     Running
calico-node-xxxxx              1/1     Running
calico-typha-xxxxx             1/1     Running
```

Then:

```bash
kubectl get tigerastatus
```

Calico should no longer report errors related to:

```text
podSubnet
```

---

# Step 18 — Basic Kubernetes Networking Test

Create a temporary Pod:

```bash
kubectl run test-pod \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 3600
```

Check:

```bash
kubectl get pod test-pod -o wide
```

You should see a Pod IP in the:

```text
192.168.x.x
```

range.

Test networking:

```bash
kubectl exec -it test-pod -- ping -c 3 8.8.8.8
```

Test DNS:

```bash
kubectl exec -it test-pod -- nslookup kubernetes.default
```

Clean up:

```bash
kubectl delete pod test-pod
```

---

# Troubleshooting Guide

## Problem 1 — `kubeadm join` says user is not root

Error:

```text
[ERROR IsPrivilegedUser]: user is not running as root
```

Use:

```bash
sudo kubeadm join ...
```

---

# Problem 2 — Worker says `kubelet.conf already exists`

Example:

```text
/etc/kubernetes/kubelet.conf already exists
Port 10250 is in use
/etc/kubernetes/pki/ca.crt already exists
```

This normally means the worker contains configuration from a previous Kubernetes cluster.

On that worker:

```bash
sudo kubeadm reset -f
```

Then:

```bash
sudo rm -rf /etc/cni/net.d
```

Restart:

```bash
sudo systemctl restart kubelet
```

Then use the **current join command**:

```bash
sudo kubeadm join ...
```

---

# Problem 3 — Calico Pods are `Running` but `0/1`

Don't immediately assume that the Pod just needs more time.

Check:

```bash
kubectl describe pod -n calico-system <pod-name>
```

Look at:

```text
Events
```

Also check:

```bash
kubectl get tigerastatus
```

And:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.podCIDR}{"\n"}{end}'
```

If Pod CIDRs are missing, check how the cluster was initialized.

The control plane should have been initialized with:

```bash
--pod-network-cidr=192.168.0.0/16
```

---

# Problem 4 — Old Calico Pod is stuck in `Terminating`

Check:

```bash
kubectl get pods -A | grep -i calico
```

If a specific old Pod is stuck terminating in a **fresh/disposable lab cluster**, you can force-delete that Pod:

```bash
kubectl delete pod <pod-name> \
  -n <namespace> \
  --grace-period=0 \
  --force
```

Then verify:

```bash
kubectl get pods -A | grep -i calico
```

**Do not make force deletion your first troubleshooting step.** First understand why the Pod is stuck.

---

# Problem 5 — Calico says `podSubnet` is missing

If:

```bash
kubectl get tigerastatus
```

shows something similar to:

```text
kubeadm configuration is missing required podSubnet field
```

check:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.podCIDR}{"\n"}{end}'
```

If the values are blank, the cluster was initialized without the required Pod network CIDR.

For a **fresh lab cluster**, the clean solution is to reset and initialize again:

```bash
sudo kubeadm reset -f
```

Clean the CNI configuration:

```bash
sudo rm -rf /etc/cni/net.d
```

Then initialize again:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=<MASTER_PRIVATE_IP> \
  --pod-network-cidr=192.168.0.0/16
```

Then configure `kubectl`, install Calico again, generate a new join command, and rejoin the workers.

---

# Problem 6 — `AlreadyExists` while installing Calico

You may see:

```text
AlreadyExists
```

for resources such as:

```text
customresourcedefinitions
tigera-operator
namespace tigera-operator
```

This does **not automatically mean the installation failed**.

Check:

```bash
kubectl get pods -n tigera-operator
```

If:

```text
tigera-operator   1/1   Running
```

then the operator already exists and is healthy.

Continue by checking:

```bash
kubectl get tigerastatus
```

---

# Problem 7 — CoreDNS is Pending

Immediately after:

```bash
kubeadm init
```

you may see:

```text
coredns   0/1   Pending
```

This is expected **before the CNI is installed**.

After Calico is successfully installed, check:

```bash
kubectl get pods -n kube-system
```

CoreDNS should become:

```text
1/1   Running
```

---

# Final Validation Checklist

Before moving to the application deployment, verify all of these.

### Nodes

```bash
kubectl get nodes
```

Expected:

```text
3 nodes
3 Ready
```

### System Pods

```bash
kubectl get pods -A
```

No important system Pod should remain unhealthy.

### Calico

```bash
kubectl get pods -n calico-system
```

Calico components should be Ready.

### Tigera

```bash
kubectl get tigerastatus
```

No critical `Degraded` state.

### Pod CIDRs

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.podCIDR}{"\n"}{end}'
```

Each node should have a Pod CIDR.

### DNS

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  --rm -it \
  -- nslookup kubernetes.default
```

### Cluster information

```bash
kubectl cluster-info
```

---

# Final Expected State

At this point you have a clean kubeadm-based Kubernetes cluster:

```text
                         AWS VPC
                            |
                +-----------+-----------+
                |                       |
          K8s Master               Worker Nodes
          10.0.1.x                 10.0.1.x
                |                   /       \
                |                  /         \
                +-----------------+-----------+
                                  |
                               Calico
                                  |
                         Pod Network
                       192.168.0.0/16
```

And:

```text
Master          Ready
Worker-01       Ready
Worker-02       Ready
```

The cluster is now ready for the **CloudVibe Voting Application** deployment.

**The most important things we learned from the troubleshooting:** configure `SystemdCgroup=true`, enable the required Linux networking/sysctl settings, initialize kubeadm with the Pod CIDR, use a stable Calico release rather than a `master` image, and reset stale worker state before rejoining a rebuilt cluster.
