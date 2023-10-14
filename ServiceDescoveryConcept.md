
# Service Discovery in Kubernetes

## Overview

In this section, we'll explore how to route traffic between different objects and make them discoverable within and outside our Kubernetes cluster. We'll introduce the concept of Kubernetes Services and how to use them to expose applications deployed using controllers like Deployments.

### What is Service?

A Service in Kubernetes defines policies by which a logical set of Pods can be accessed. It enables communication between various components of an application and different applications, allowing us to connect applications with other applications or users.

## Service Configuration

Here's an example manifest for a Kubernetes Service:

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

- `apiVersion` is set to `v1`.
- `kind` is always `Service`.
- In the `metadata` field, we specify the name of the Service.

## Types of Services

There are four different types of Services in Kubernetes:

1. **NodePort**: This type makes internal Pod(s) accessible on a port on the node on which the Pod(s) are running.

2. **ClusterIP**: It exposes the Service on a certain IP inside the cluster. This is the default type of Service.

3. **LoadBalancer**: This type exposes the application externally using the load balancer provided by the cloud provider.

4. **ExternalName**: It points to a DNS rather than a set of Pods, unlike the other types that use label selectors.

### NodePort Service

A NodePort Service exposes the application on the same port on all nodes in the cluster. It spans across all nodes and exposes Pods on a specific port. This way, the application can be accessed from outside the Kubernetes cluster using the IP/port combination `<NodeIP>:<NodePort>`.

Here's a sample NodePort Service configuration:

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

- `targetPort`: The port where the application running on the Pods is exposed.
- `port`: The port of the Service itself.
- `nodePort`: The port on the node used to access the Service.

Kubernetes Services are essential for making Pods discoverable and accessible within the cluster and externally, allowing different components and applications to communicate effectively.
```

You can save the above content in a markdown file with a `.md` extension and then upload it to your GitHub repository.
