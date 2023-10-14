
# Service Discovery in Kubernetes

## Overview

In this section, we will explore how to route traffic between various Kubernetes objects and make them discoverable from both within and outside our cluster. We will introduce Kubernetes Services and explain how to use them to expose applications deployed using controllers such as Deployments. By the end, you will be able to make your application accessible to the external world and understand the different types of Services available.

### Problem Statement

Each Kubernetes Pod gets its IP address, which can change in case of Pod relaunch. To ensure reliability, we use Deployments to maintain a fixed number of Pods. However, the changing IP addresses of Pods create the need to make Pods discoverable within the cluster.

### Solution: Kubernetes Services

Kubernetes Services enable communication between different components of our application, as well as between different applications. Services help connect applications with other applications or users.

## What is a Service?

A Service defines policies by which a logical set of Pods can be accessed. It allows the discovery and access of Pods, either within the cluster or externally.

## Service Configuration

Here is an example manifest for a Kubernetes Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-sample-service
spec:
  ports:
    - port: 80
      targetPort: 80
  selector:
    key: value
```

- `apiVersion` is set to v1.
- `kind` is always "Service."
- In the `metadata` field, we specify the name of the Service.

## Types of Services

There are four different types of Services in Kubernetes:

1. **NodePort**: Makes internal Pod(s) accessible on a port on the node where the Pod(s) are running.

2. **ClusterIP**: Exposes the Service on an IP address inside the cluster (default Service type).

3. **LoadBalancer**: Exposes the application externally using the cloud provider's load balancer.

4. **ExternalName**: Points to a DNS rather than a set of Pods. It doesn't use selectors.

### NodePort Service

A NodePort Service exposes the application on the same port on all nodes in the cluster. Pods may be running on multiple nodes, and the Service spans across all nodes, making the application accessible via `<NodeIP>:<NodePort>`.

Example NodePort Service configuration:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  ports:
    - targetPort: 80
      port: 80
      nodePort: 32023
  selector:
    app: nginx
    environment: production
```

- `targetPort`: Port where the application on Pods is exposed.
- `port`: Port of the Service itself.
- `nodePort`: Port on the node to access the Service.

#### Lab 1: Creating a NodePort Service with Nginx Containers

- Create a Deployment for Nginx.
- Create a NodePort Service for Nginx.
- Verify the Service and access Nginx.

### ClusterIP Service

ClusterIP Service exposes the application on a specific IP address accessible only from inside the cluster. It's suitable for communication between different Pods within the cluster.

Example ClusterIP Service configuration:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  ports:
    - targetPort: 80
      port: 80
  selector:
    app: nginx
    environment: production
```

- `type`: Set to ClusterIP.
- `targetPort` and `port` specify the port of the Pods and the port for the Service.

#### Lab 2: Creating a ClusterIP Service with Nginx Containers

- Create a Deployment for Nginx.
- Create a ClusterIP Service for Nginx.
- Verify the Service and access Nginx.

### LoadBalancer Service

A LoadBalancer Service exposes the application externally using the cloud provider's load balancer. It assigns an external IP address to the Service. Configuration may vary based on the cloud provider.

Example LoadBalancer Service configuration (simplified):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: loadbalancer-service
spec:
  type: LoadBalancer
  ports:
    - targetPort: 8080
      port: 80
  selector:
    app: nginx
    environment: production
```

- Each cloud provider requires specific annotations for LoadBalancer configuration.

## Conclusion

Kubernetes Services play a crucial role in making applications and Pods discoverable and accessible within a cluster and from external sources. Understanding the types and configurations of Services is essential for effective service discovery and communication in Kubernetes.
