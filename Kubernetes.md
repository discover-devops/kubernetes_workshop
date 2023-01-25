Overview of Kubernetes



In this talk I'm going to talk about the a brief introduction of Kubernetes and set-up a single node minikube Kubernetes cluster. first we will understand how kubernetes works.

Setting up our lab:
We ca install Kubernetes by many ways:

1: Virtual Machine/Physical Server/bare metal: Kubeadm is a better choice.

https://kubernetes.io/docs/reference/setup-tools/kubeadm/

However, you can also consider to use 

2: For any public cloud, there own managed Kubernetes services (control plane is managed by the Cloud vendor), EKS, AKS, OKE, GKE etc ..
Also, you can use Kops (AWS) and kubespray

3: For Lab and learning purpose, you can use minikube (we will use for our lab).

So, what is minikube?
https://minikube.sigs.k8s.io/

minikube quickly sets up a local Kubernetes cluster on macOS, Linux, and Windows.

Let's discuss about the Kubernetes Architecture. 

Kubernetes main components are:

API server
etcd
Controller manager
Scheduler
Kubelet

Besides these core components, there are few more components. The thing where our applications runs in form for containers or 
group of containers are known as Pods.

The resources in Kubernetes, such as Pods, Services, Deployments etc. etc... are known as API object.

A Kubernetes object is a "record of intent"--once you create the object, the Kubernetes system will constantly work to ensure 
that object exists. 

By creating an object, you're effectively telling the Kubernetes system what you want your cluster's workload to look like; 
this is your cluster's desired state.

To work with Kubernetes objects--whether to create, modify, or delete them--you'll need to use the Kubernetes API.

So, in nut-shell a kubernetes API object describes how a certain resource should be honored in Kubernetes. 

Mostly we use a human readable file to define the object and then use a tool like kubectl to parse it and hand it over to a 
Kubernetes API server.





Kubernetes Architecture :

A Kubernetes cluster has two main components:

1: Master Nodes also known as control plane or the manager who controls and manages the the whole Kubernetes system.
2: Worker Nodes where the actual application runs.



![image-20220103141921055](C:\Users\rahchaub\AppData\Roaming\Typora\typora-user-images\image-20220103141921055.png)



**The Control Plane / The master/ The manager** 

Control plane it consists of multiple components that can run on a single master node or be split across multiple nodes and 
replicated to ensure high availability. 

The main components are:  

- The **Kubernetes API Server**, it acts like a gateway, through which the end-user and the other Control Plane components communicate with Kubernetes cluster.

- The **Scheduler**, as the name suggest, this schedules the end users application and selects a worker node for deployment (scheduler is only responsible for deciding the node, it doesn't actually place the apps (pod) on the nodes).

- The **Controller Manager** which performs cluster-level functions, such as replicating components, keeping track of worker nodes, handling node failures, and so on. It is having combination of multiple controllers such as node-controllers, which take care of node counts.


- **etcd**, a reliable distributed data store that persistently stores the cluster configuration.



The Worker Node/ Data Plane

The worker nodes are the machines that run end-users containerized applications. Each worker nodes has the following components:


- **Container run-time**: Docker/rkt/containerd/cri-o runtime, which runs the end-users containers.
- The **Kubelet**, which talks to the API server and manages containers on its node.
- The Kubernetes Service Proxy (**kube-proxy**), which does two things, first load-balances network traffic between application components and second to ensure that inbound traffic to a Service endpoint can be routed properly. 



![img](https://cdn-images-1.medium.com/max/1200/1*lasosXDxyoktgKHYT91P5Q.png)

Why Kubernetes?

- If your Application is having many containers, then Kubernetes is the tool to manage it.
- Helping developer to focus on there coding instead of handling the infra management.



What we can see from the Above diagram:

The API server can talk to almost every other component (except the container runtime).
API server can interact ONLY with etcd. The API server also scrutinizes incoming requests and writes API objects into the backend storage (etcd).
API server performs authentication, authorization, and auditing of any incoming request.
There is no explicit connection among the components except API server. I.E. the controller manager doesn't talk to the scheduler, nor does the kubelet talk to kube-proxy. Instead, they communicate implicitly via the API server.

Apart from this, there is one more important component of the K8s Architecture, known as CNI (**Container Network Interface**).

Why CNI?
How pod communicate with each other.
How Pod communicate with the host machine.
How 2 different nodes communicate with other.
How pod running on different nodes communicate with each other.
Kubernetes chooses to solve those problems by defining a specification called the Container Network Interface (CNI).



