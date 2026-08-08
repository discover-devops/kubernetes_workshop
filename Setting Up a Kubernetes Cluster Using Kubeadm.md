# Setting Up a Kubernetes Cluster Using Kubeadm

## A Step by Step Runbook

This guide will help you build your own working Kubernetes cluster from scratch. You do not need to be an expert to follow this. Every step is written out fully. Every command is ready to copy and paste. Wherever a new idea shows up, it is explained in plain language before you use it.

---

## Part 1: The Big Picture

Before we touch a single command, let us understand what we are actually building.

A Kubernetes cluster is made up of machines. These machines fall into two roles.

One machine is the Control Plane. Think of the control plane as the manager or the brain of the cluster. It does not run your actual applications. Instead it makes decisions. It keeps track of what should be running, where it should run, and whether it is healthy.

The other machines are called Worker Nodes. These are the machines where your actual applications run inside containers. Workers take instructions from the control plane and carry them out.

In this runbook we will build a cluster with:

- One Control Plane machine
- Two Worker Node machines

The tool we will use to build this cluster is called kubeadm. Kubeadm is an official Kubernetes tool that automates most of the setup work. Without kubeadm you would need to manually configure many components. Kubeadm handles that for you and gets a working cluster up quickly.

Here is the overall plan we will follow, in order.

1. Create three machines. This runbook shows you how to do this on AWS EC2. It also explains what changes if you are using two plain Linux machines instead of EC2.
2. Prepare every machine the same way. This includes turning off certain settings that Kubernetes does not allow, and installing the container runtime and the Kubernetes tools.
3. Turn one machine into the Control Plane using kubeadm init.
4. Install a Pod Network add on so that containers on different machines can talk to each other.
5. Join the two Worker Nodes to the Control Plane using kubeadm join.
6. Verify that everything is working.
7. Understand how traffic actually flows into your cluster once it is running.

Once this is done, you will have a real, working Kubernetes cluster that you built yourself, piece by piece.

---

## Part 2: Concepts You Should Know Before Starting

### What is a Container Runtime

Kubernetes does not run containers by itself. It depends on a separate piece of software called a container runtime to actually start and stop containers. In this runbook we will use containerd, which is the most common and recommended runtime today.

### What is kubelet

Kubelet is an agent that runs on every machine in the cluster, including the control plane. It is the component that actually talks to the container runtime and makes sure the containers that are supposed to be running are actually running.

### What is kubeadm

Kubeadm is the setup tool. You use it once to create the cluster, and once on each worker node to join that node to the cluster. After the cluster is created, you do not use kubeadm on a daily basis. You use kubectl instead.

### What is kubectl

Kubectl is the command line tool you use to talk to your cluster once it exists. You will use kubectl constantly after setup is complete.

### What is a Pod Network or CNI

Every pod in your cluster needs an IP address, and pods on different machines need to be able to talk to each other over the network as if they were all on the same network, even though they are physically on different machines. A CNI, which stands for Container Network Interface, is the add on that makes this possible. Without installing a CNI, your cluster will not work properly. In this runbook we will install Calico as our CNI.

### How Traffic Actually Reaches Your Application

This is a common point of confusion, so let us walk through it clearly.

When traffic needs to reach an application running inside your cluster, it typically enters through one of these three doors.

- A Load Balancer, which is usually provided by your cloud provider
- A NodePort, which opens a specific port directly on every worker node
- An Ingress, which is a smarter routing layer that sits in front of your services and reads the web address being requested to decide where to send the traffic

Once traffic gets past that first door, it needs to reach the correct Pod. This is where a component called kube proxy comes in. Kube proxy runs on every node and it is responsible for making sure that traffic aimed at a Service actually reaches one of the healthy Pods behind that Service. It does this by maintaining a set of rules inside the machine, most commonly using something called iptables, although newer setups can use a mode called IPVS instead.

So the correct way to think about it is this. Traffic first enters through Ingress, Load Balancer, or NodePort. Then kube proxy, using the rules it maintains on each node, directs that traffic to the correct Pod. Ingress and kube proxy are not competing with each other, they are two different layers working together. Ingress decides which Service should handle a request, and kube proxy decides which specific Pod behind that Service actually receives it.

---

## Part 3: Prerequisites

Before starting, make sure you have the following.

- Three Linux machines. Ubuntu 22.04 is used in this runbook. If you are using AWS EC2, this runbook will show you how to launch them. If you already have two Linux machines available to you, whether on a laptop, a home server, or any other provider, you can skip the EC2 launch step and go directly to Part 5.
- Each machine should have at least 2 CPU cores and at least 2 GB of memory. This is the minimum. More is better if available.
- All three machines must be able to reach each other over the network. If you are on EC2 this means they must be in the same VPC and the same subnet is recommended for simplicity.
- A way to connect to each machine using SSH.

---

## Part 4: Launching the Machines on AWS EC2

If you already have your Linux machines ready, skip ahead to Part 5.

Step 1. Log in to the AWS Console and go to the EC2 service.

Step 2. Click Launch Instance.

Step 3. Choose the Ubuntu Server 22.04 LTS image.

Step 4. Choose an instance type. t3.medium is a good choice for this exercise since Kubernetes needs a reasonable amount of memory.

Step 5. Create or choose an existing key pair. You will need this file to connect to your machines using SSH. Keep it safe.

Step 6. Under Network Settings, make sure all three machines will be launched into the same VPC and the same subnet.

Step 7. Configure the Security Group. This is one of the most important steps. A Security Group is basically a firewall that controls what traffic is allowed in and out of your machine. Add the following inbound rules so that your cluster components can talk to each other.

- SSH, port 22, source your own IP address
- All Traffic, allowed from within the same Security Group. This is the simplest way to let your three machines talk freely to each other for learning purposes. You can tighten this later once you understand the specific ports each component needs.

Step 8. Set Number of Instances to 3, so you launch the control plane and both worker nodes in one go.

Step 9. Click Launch Instance.

Step 10. Once the instances are running, note down the Private IP address of each machine. You will need these later. Also give each machine a clear name in the AWS Console, such as k8s-control-plane, k8s-worker-1, and k8s-worker-2, so you do not get confused later.

---

## Part 5: Preparing Every Machine

The steps in this part must be done on all three machines. This includes the control plane and both worker nodes. Do not skip this on any machine.

Connect to each machine using SSH before starting.

### Step 5.1 Update the machine

```
sudo apt update
sudo apt upgrade -y
```

### Step 5.2 Set a unique hostname on each machine

This step is optional but strongly recommended, since it makes it much easier to tell your machines apart later.

On the control plane machine, run

```
sudo hostnamectl set-hostname k8s-control-plane
```

On the first worker node, run

```
sudo hostnamectl set-hostname k8s-worker-1
```

On the second worker node, run

```
sudo hostnamectl set-hostname k8s-worker-2
```

### Step 5.3 Disable swap

Kubernetes requires swap to be turned off. Swap is a feature that lets a machine use disk space as extra memory when it runs low on real memory. Kubernetes disables this requirement because it needs predictable memory behavior to work correctly.

Run this on every machine.

```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

The first command turns swap off immediately. The second command comments out the swap entry in a configuration file called fstab, so that swap does not turn itself back on the next time the machine restarts.

### Step 5.4 Load required kernel modules

Kubernetes networking depends on certain features being active inside the Linux kernel. Run the following on every machine.

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### Step 5.5 Configure required networking settings

Run the following on every machine.

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

This tells the machine to allow traffic to be properly seen and forwarded between containers, which Kubernetes networking depends on.

### Step 5.6 Install containerd, the container runtime

Run the following on every machine.

```
sudo apt install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

Now open the configuration file and make one important change.

```
sudo nano /etc/containerd/config.toml
```

Find the line that says

```
SystemdCgroup = false
```

Change it to

```
SystemdCgroup = true
```

This setting tells containerd to manage resources using the same system, called systemd, that Kubernetes itself expects. If this is left as false, your kubelet may fail to start correctly later.

Save the file and exit, then restart containerd.

```
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Step 5.7 Install kubeadm, kubelet, and kubectl

Run the following on every machine.

```
sudo apt install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

A quick note. This runbook installs version 1.30. Kubernetes releases new versions regularly. If you want the latest stable version instead, visit the official Kubernetes release page linked in the References section at the end of this document, and simply replace v1.30 in the commands above with the version you want.

The last command, apt-mark hold, prevents these tools from being accidentally upgraded by a routine system update, which could break your cluster unexpectedly.

At this point, Part 5 is complete on all three machines. Every machine now has the container runtime and the Kubernetes tools installed and ready.

---

## Part 6: Creating the Control Plane

Do the steps in this part only on the control plane machine.

### Step 6.1 Initialize the cluster

```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

The pod-network-cidr value is a range of IP addresses that will be handed out to your pods. We are specifying 192.168.0.0/16 here because this is the range expected by Calico, the pod network add on we will install shortly. If you choose a different CNI later, this value may need to change, so it is worth remembering that this number is tied to your networking choice, not a random setting.

This command will take a few minutes to run. When it finishes successfully, it will print an important block of text that includes a kubeadm join command. Copy this entire block somewhere safe right now. You will need it in Part 8 to connect your worker nodes. If you lose it, do not worry, Part 8 also explains how to generate a new one.

### Step 6.2 Allow your regular user to run kubectl

Kubernetes stores the credentials needed to talk to the cluster in a file. By default only the root user can read it. Run the following as the same user you plan to use day to day.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Step 6.3 Confirm the control plane is up

```
kubectl get nodes
```

At this stage you will see only one node, the control plane, and its status will likely show as NotReady. This is expected and completely normal. It is NotReady because we have not installed the Pod Network yet. Kubernetes will not mark a node as Ready until networking is properly set up. We fix this in the next part.

---

## Part 7: Installing the Pod Network

Do this step only on the control plane machine.

We are using Calico as our CNI, which is one of the most widely used and reliable options.

```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

Give it a couple of minutes to fully start up, then check the status again.

```
kubectl get nodes
```

The control plane should now show as Ready. You can also check that all the networking pods came up correctly.

```
kubectl get pods -n kube-system
```

Every pod in this list should eventually show as Running.

---

## Part 8: Joining the Worker Nodes

Do the steps in this part on both worker node machines, k8s-worker-1 and k8s-worker-2.

### Step 8.1 Run the join command

Use the exact join command that was printed at the end of Step 6.1. It will look similar to this, though your actual token and hash will be different.

```
sudo kubeadm join 10.0.1.15:6443 --token abc123.xxxxxxxxxxxxxxxx \
  --discovery-token-ca-cert-hash sha256:1111111111111111111111111111111111111111111111111111111111
```

Run this command on k8s-worker-1, and once it completes, run the same command on k8s-worker-2.

### Step 8.2 If you lost the original join command

Tokens expire after 24 hours for security reasons. If yours has expired or you simply lost it, go back to the control plane machine and run

```
sudo kubeadm token create --print-join-command
```

This will print a brand new, valid join command that you can use on your worker nodes.

---

## Part 9: Verifying the Cluster

Go back to the control plane machine and run

```
kubectl get nodes
```

You should now see three machines listed, your control plane and both worker nodes, and every single one of them should show a status of Ready. If any node is not showing Ready yet, give it a minute and check again, since it can take a short while for everything to settle.

You can also confirm that basic system pods are healthy across the whole cluster by running

```
kubectl get pods -A
```

All pods listed should eventually show a status of Running.

Congratulations, at this point you have a fully working three node Kubernetes cluster that you built entirely by hand using kubeadm.

---

## Part 10: A Quick Test Deployment

To confirm your cluster can actually run an application, try this simple test.

```
kubectl create deployment nginx-test --image=nginx --replicas=2
kubectl get pods -o wide
```

The -o wide flag shows you which node each pod landed on. You should see the two nginx pods spread across your worker nodes. This is Kubernetes making the same kind of decision we described back in Part 1, the control plane deciding where your application should actually run.

You can clean up this test once you are done looking at it.

```
kubectl delete deployment nginx-test
```

---

## Part 11: If You Are Using Two Plain Linux Machines Instead of EC2

Everything in Parts 5 through 10 works exactly the same way regardless of where your machines are running. Kubernetes and kubeadm do not care whether your machine is an EC2 instance, a machine sitting on your desk, or a virtual machine on your laptop. The only two things that change are these.

First, in Part 4, you do not need to launch anything on AWS. You simply use the two Linux machines you already have, plus you would need a third machine to act as the control plane, since this runbook is built for one control plane and multiple workers. If you genuinely only have two machines total, one of them becomes the control plane and the other becomes your only worker node, and everywhere this guide says worker-2, you simply skip it.

Second, make sure your machines can reach each other over the network. On EC2 this is handled by putting them in the same VPC and subnet. On your own machines, this usually means connecting them to the same router or the same local network, and making sure no firewall is blocking traffic between them.

Everything else, every command from Part 5 onward, is identical.

---

## Part 12: Common Problems and How to Fix Them

Problem: kubectl get nodes shows the node as NotReady even after installing Calico.

This is almost always a networking add on issue. Run kubectl get pods -n kube-system and look for any Calico pods that are not Running. Often this resolves itself within a few minutes, but if it does not, it usually means the pod-network-cidr you used in kubeadm init does not match what Calico expects.

Problem: kubeadm join fails with a connection timeout.

This almost always means the worker node cannot reach the control plane over the network. Double check that your Security Group, or your local firewall if you are not using EC2, is allowing traffic between the machines.

Problem: kubelet fails to start after installation.

Check that you correctly changed SystemdCgroup to true in the containerd configuration file in Step 5.6, and that you restarted containerd afterward.

Problem: The join token has expired.

Follow Step 8.2 to generate a new join command from the control plane.

---

## Part 13: Cleaning Up

If you want to tear down what you built, here is how.

On any worker node, to remove it from the cluster

```
sudo kubeadm reset -f
```

On the control plane, to fully remove the cluster setup

```
sudo kubeadm reset -f
sudo rm -rf $HOME/.kube
```

If you are using EC2 and want to stop incurring any cost, simply terminate the three instances from the AWS Console once you are done practicing.

---

## Part 14: Check Your Understanding

Before moving on, make sure you can answer these questions in your own words.

1. What is the difference between the role of the control plane and the role of a worker node.
2. Why does Kubernetes require swap to be turned off.
3. What job does containerd do that Kubernetes itself does not do.
4. What problem does installing a CNI like Calico actually solve.
5. Once traffic enters through an Ingress, a Load Balancer, or a NodePort, what component takes over and gets that traffic to the correct Pod, and what mechanism does it typically use to do this.
6. What command do you run on a worker node to have it join an existing cluster, and where does that command come from.

---

## References

Official Kubernetes kubeadm installation guide
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

Official kubeadm cluster creation guide
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

Official list of current Kubernetes releases
https://kubernetes.io/releases/

Official Calico installation guide
https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

Official containerd documentation
https://containerd.io/docs/

AWS EC2 documentation
https://docs.aws.amazon.com/ec2/
