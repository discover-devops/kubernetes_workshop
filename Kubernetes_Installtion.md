## **<u>Kubernetes Installation</u>** <u>using Kubeadm</u>



Installation of Kubernetes using Kubeadm is simple 4 step process:

![image-20220108154818088](C:\Users\rahchaub\AppData\Roaming\Typora\typora-user-images\image-20220108154818088.png)

##### <u>Step1: Operating System Changes/Configuration updates:</u>

This is 2 node Kubernetes Cluster and below specifications: 
**MasterNode**  10.0.0.8   Linux (ubuntu 18.04)
**WorkerNode1**  10.0.0.9 Linux (ubuntu 18.04)



1.1: Update the OS and Add master and worker nodes entry in **/etc/hosts** file

> $ apt update -y && apt upgrade -y
> $ cp -p /etc/hosts  /etc/hosts.ORIG
> $ echo "MasterNode  10.0.0.8" | tee -a /etc/hosts
> $ echo "WorkerNode1  10.0.0.9" | tee -a /etc/hosts

1.2: Disable Swap Memory

> $ sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
> $ swapoff -a

1.3: Configure Firewall and Networking parameter's 

>  # Load Modules
>
> $ sudo modprobe overlay
> $ sudo modprobe br_netfilter
>
> ###### Set system configurations for Kubernetes networking:
>
> cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
> net.bridge.bridge-nf-call-iptables = 1
> net.ipv4.ip_forward = 1
> net.bridge.bridge-nf-call-ip6tables = 1
> EOF
>
> ###### Apply new settings:
> sudo sysctl --system



##### <u>Step2: Install Kubelet, Kubeadm and Kubectl:</u>



> ###### Install dependency packages:
> apt-get update &&  apt-get install -y apt-transport-https curl
>
> ###### Download and add GPG key:
> curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
>
> ###### Add Kubernetes to repository list:
> cat <<EOF |  tee /etc/apt/sources.list.d/kubernetes.list
> deb https://apt.kubernetes.io/ kubernetes-xenial main
> EOF
>
> ###### Update package listings:
> apt-get update
>
> ###### Install Kubernetes packages:
> apt-get install -y kubelet kubeadm kubectl
>
> ###### Turn off automatic updates:
> apt-mark hold kubelet kubeadm kubectl
>
> ###### Verify the Versions
>
> kubectl version --client && kubeadm version

##### <u>Step3: Install Container Runtime</u>

3.1: Docker as Container Runtime

> ###### Install the Docker 
>
> $ apt update
> $ apt install -y curl gnupg2
> $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
> $ add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
> $ apt update
> $ apt install -y containerd.io docker-ce
>
> ###### Configure the Docker daemon, in particular to use systemd for the management of the container’s cgroups.
>
> $ mkdir /etc/docker
> cat <<EOF | sudo tee /etc/docker/daemon.json
> {
>   "exec-opts": ["native.cgroupdriver=systemd"],
>   "log-driver": "json-file",
>   "log-opts": {
>     "max-size": "100m"
>   },
>   "storage-driver": "overlay2"
> }
> EOF
>
> ###### Restart Docker and enable on boot:
>
> $ systemctl enable docker
> $ systemctl daemon-reload
> $ systemctl restart docker
>
> $ systemctl status docker



##### **OR** 

##### **3.1: Containerd as Runtime**

> ###### Create configuration file for containerd:
>
> $ cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
> overlay
> br_netfilter
> EOF
>
> ###### Create default configuration file for containerd:
> $ mkdir -p /etc/containerd
>
> ###### Generate default containerd configuration :
> $ containerd config default | tee /etc/containerd/config.toml
>
> ###### Restart containerd to ensure new configuration file usage:
> $ systemctl restart containerd
>
> ###### Verify that containerd is running:
> $ systemctl status containerd



##### <u>Step4: Create Kubernetes Cluster and Join Worker Nodes</u>

> ###### Make sure that br_netfilter module is loaded:
>
> $ lsmod | grep br_netfilter
>
> ###### Start Kubelet
>
> $ systemctl enable kubelet
>
> ###### Create Cluster
>
> $ kubeadm init  --apiserver-advertise-address=10.0.0.8 --pod-network-cidr=192.168.0.0/16 
>
> ###### Set kubectl access:
> $ mkdir -p $HOME/.kube
> $ cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
> $ chown $(id -u):$(id -g) $HOME/.kube/config
>
> ###### Install Calico Networking:
>
> $ kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
>
> ###### Check status of the control plane node:
> $ kubectl get nodes
>
> ###### In the Control Plane Node, create the token and copy the kubeadm join command . The join command can also be found in the output from kubeadm init command.
>
> $ 	kubeadm token create --print-join-command
>
> ###### On Worker Node run the below Command to join the Cluster
>
> $ kubeadm join master:6443 --token <> --discovery-token-ca-cert-hash <> --control-plane
>
> ###### Check the Cluster Status
>
> $ kubectl cluster-info
>
> $ kubectl get nodes -o wide
>
> $ watch kubectl get pods --all-namespaces
>
> ###### Run a test Pod
>
> $ kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
>
> $ kubectl get deployments
>
> $ kubect get pods



##### Troubleshooting 

You may need below steps to troubleshoot the installation stesp:

> ###### Make sure that below Ports are open at your Firewall ports ot Cloud Security groups and NACLs
>
> 6443, 10250, 10259, 10257, 2379, 2380, 30000-32767
>
> ###### At Master Nodes, open the below ports over OS firewall
>
> ###### UFW
>
> Master Nodes:
> sudo ufw allow 6443/tcp
> sudo ufw allow 10250/tcp
> sudo ufw allow 10259/tcp
> sudo ufw allow 10257/tcp
> sudo ufw allow 2379/tcp
> sudo ufw allow 2380/tcp
>
> Worker Nodes:
> sudo ufw allow 30000:32767/tcp
> sudo ufw allow 10250/tcp
>
> sudo ufw disable 
> sudo ufw enable
>
> ###### iptables:
> sudo iptables -S
> sudo iptables -L
>
> Master:
> sudo iptables -A INPUT -p tcp -s <CIDR> --dport 6443 -j ACCEPT
> sudo iptables -A INPUT -p tcp -s <CIDR> --dport 10250 -j ACCEPT
> sudo iptables -A INPUT -p tcp -s <CIDR> --dport 10259 -j ACCEPT
> sudo iptables -A INPUT -p tcp -s <CIDR> --dport 10257 -j ACCEPT
> sudo iptables -A INPUT -p tcp -s <CIDR> --dport 2379 -j ACCEPT
> sudo iptables -A INPUT -p tcp -s <CIDR> --dport 2380 -j ACCEPT
>
> Worker:
> sudo iptables -A INPUT -p tcp -s <CIDR>  --dport 10250 -j ACCEPT
> sudo iptables -A INPUT -p tcp -s <CIDR>  --dport 30000:32767 -j ACCEPT
>
> sudo iptables-save
> service iptables stop
> service iptables start 

##### References:

[1]: https://kubernetes.io/docs/setup/#production-environment
[2]: https://docs.docker.com/engine/install/#server

