Kubernetes, often abbreviated as K8s, is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications. It was originally developed by Google and is now maintained by the Cloud Native Computing Foundation (CNCF). Kubernetes provides a framework for automating the deployment, scaling, and management of containerized applications, allowing developers to focus on building and running applications without worrying about the underlying infrastructure.


![image](https://github.com/discover-devops/kubernetes_workshop/assets/53135263/8fec9558-a887-4966-9b64-857c00a90d99)



**Key Concepts:**

1. **Node:** A node is a physical or virtual machine that runs containerized applications. It could be a VM (Virtual Machine) or a physical server.

2. **Pod:** The smallest and simplest unit in the Kubernetes object model. A pod represents a single instance of a running process in a cluster and can contain one or more containers.

3. **Container:** A lightweight, standalone, and executable software package that includes everything needed to run a piece of software, including the code, runtime, libraries, and dependencies.

4. **Deployment:** A resource object in Kubernetes that provides declarative updates to applications. A deployment allows you to describe an application’s life cycle, such as which images to use for the app, the number of pod replicas, and the way to update them.

5. **Service:** A Kubernetes resource that defines a logical set of pods and a policy by which to access them. Services enable network communication between different sets of pods.

6. **Namespace:** A way to divide cluster resources between multiple users (via resource quota), teams, or projects. It's useful when there are multiple users or teams sharing a Kubernetes cluster.

**Kubernetes Architecture:**

Kubernetes follows a master-worker architecture, consisting of the following components:

1. **Master Node:**
   - **API Server:** Serves as the entry point for the Kubernetes control plane. It validates and processes requests, and then updates the corresponding objects like pods, services, etc.
   - **Controller Manager:** Ensures that the desired state of the cluster matches the actual state by controlling various controllers like node controller, replication controller, etc.
   - **Scheduler:** Assigns nodes to newly created pods based on resource requirements and other constraints.
   - **etcd:** A distributed key-value store that stores the configuration data of the cluster, representing the current state of the entire system.

2. **Node (Minion) Node:**
   - **Kubelet:** An agent that runs on each node and is responsible for ensuring that the containers are running in a pod.
   - **Container Runtime:** The software responsible for running containers. Kubernetes supports various container runtimes, such as Docker, containerd, and others.
   - **Kube Proxy:** Maintains network rules on nodes and performs connection forwarding.

3. **Add-ons:**
   - Additional components and features that enhance the Kubernetes cluster, such as DNS for service discovery, dashboard for web-based management, and others.

In summary, Kubernetes orchestrates the deployment and management of containerized applications, providing a scalable and resilient infrastructure for running modern, cloud-native applications. The master node controls and manages the overall state of the cluster, while the worker nodes execute and run the containers.
