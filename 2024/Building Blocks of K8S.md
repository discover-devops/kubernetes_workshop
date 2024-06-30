### Tutorial: Understanding Pods, Replica Sets, and Deployments in Kubernetes

Kubernetes provides various constructs to manage containerized applications. Let's explore the differences between Pods, Replica Sets, and Deployments and how to use them with the NGINX image.

#### 1. Pods

A Pod is the smallest and simplest Kubernetes object. It represents a single instance of a running process in your cluster.

**YAML for launching a Pod using the NGINX image:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

#### 2. Replica Sets

A ReplicaSet ensures that a specified number of pod replicas are running at any given time. It is used primarily by Deployments as a mechanism to orchestrate pod scaling.

**YAML for launching a ReplicaSet using the NGINX image:**
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

#### 3. Deployments

A Deployment provides declarative updates for Pods and ReplicaSets. You describe a desired state in a Deployment object, and the Deployment Controller changes the actual state to the desired state at a controlled rate.

**YAML for launching a Deployment using the NGINX image:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

### Which is Better: Pod, ReplicaSet, or Deployment?

**1. Pods:**
   - **Use Case:** Useful for simple use cases like single-instance applications, testing, and debugging.
   - **Limitations:** Lack of scalability and fault tolerance. If a Pod fails, it needs to be recreated manually.

**2. ReplicaSets:**
   - **Use Case:** Ensures a specified number of replicas are running. Suitable for applications needing high availability and redundancy.
   - **Limitations:** Manages replicas but does not support rolling updates. Typically used as a backend for Deployments.

**3. Deployments:**
   - **Use Case:** Provides declarative updates, self-healing, and rollbacks. Suitable for production-grade applications requiring updates, scaling, and high availability.
   - **Advantages:** Rolling updates, rollbacks, and scaling.

### Step-by-Step Guide to Launch the Application

1. **Create a Pod:**
   ```sh
   kubectl apply -f pod-nginx.yaml
   ```

2. **Create a ReplicaSet:**
   ```sh
   kubectl apply -f replicaset-nginx.yaml
   ```

3. **Create a Deployment:**
   ```sh
   kubectl apply -f deployment-nginx.yaml
   ```

4. **Verify Resources:**
   ```sh
   kubectl get pods
   kubectl get rs
   kubectl get deployments
   ```

5. **Check Logs and Status:**
   ```sh
   kubectl logs <pod-name>
   kubectl describe pod <pod-name>
   kubectl describe rs <replicaset-name>
   kubectl describe deployment <deployment-name>
   ```

6. **Scaling a Deployment:**
   ```sh
   kubectl scale deployment nginx-deployment --replicas=5
   ```

By using Deployments, you get advanced features like rolling updates and rollbacks, making them the preferred choice for most production applications.
