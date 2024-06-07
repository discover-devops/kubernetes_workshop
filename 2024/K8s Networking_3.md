## In-Depth Guide to Kubernetes Networking

### Table of Contents
1. [Kubernetes networking requirements](#kubernetes-networking-requirements)
2. [How Linux network namespaces work in a pod](#how-linux-network-namespaces-work-in-a-pod)
3. [The pause container creates the network namespace in the pod](#the-pause-container-creates-the-network-namespace-in-the-pod)
4. [The pod is assigned a single IP address](#the-pod-is-assigned-a-single-ip-address)
5. [Inspecting pod to pod traffic in the cluster](#inspecting-pod-to-pod-traffic-in-the-cluster)
6. [The pod network namespace is connected to an ethernet bridge](#the-pod-network-namespace-is-connected-to-an-ethernet-bridge)
7. [Tracing pod to pod traffic on the same node](#tracing-pod-to-pod-traffic-on-the-same-node)
8. [Tracing pod to pod communication on different nodes](#tracing-pod-to-pod-communication-on-different-nodes)
9. [The Container Network Interface - CNI](#the-container-network-interface---cni)
10. [Inspecting pod to service traffic](#inspecting-pod-to-service-traffic)
11. [Intercepting and rewriting traffic with Netfilter and Iptables](#intercepting-and-rewriting-traffic-with-netfilter-and-iptables)
12. [Inspecting responses from services](#inspecting-responses-from-services)

### Kubernetes networking requirements

Before diving into the details on how packets flow inside a Kubernetes cluster, let's first clear up the requirements for a Kubernetes network.

The Kubernetes networking model defines a set of fundamental rules:

1. A pod in the cluster should be able to freely communicate with any other pod without the use of Network Address Translation (NAT).
2. Any program running on a cluster node should communicate with any pod on the same node without using NAT.
3. Each pod has its own IP address (IP-per-Pod), and every other pod can reach it at that same address.

These requirements describe the properties of the cluster network in general terms. Implementing these rules involves solving several challenges, such as ensuring container-to-container communication within a pod, inter-pod communication, service reachability, and external traffic handling.

This article will focus on the first three points, starting with intra-pod networking or container-to-container communication.

### How Linux network namespaces work in a pod

Consider a pod with an Nginx container and another with BusyBox:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
    - name: container-1
      image: busybox
      command: ['/bin/sh', '-c', 'sleep 1d']
    - name: container-2
      image: nginx
```

When deployed, the following happens:

- The pod gets its own network namespace on the node.
- An IP address is assigned to the pod, and the ports are shared between the two containers.
- Both containers share the same networking namespace and can see each other on localhost.
- The network configuration happens rapidly in the background.

In Linux, network namespaces are isolated, logical spaces that can be configured with their own networking rules and resources. The physical network interface holds the root network namespace, and the physical interface processes all real packets in the end. Virtual interfaces created from the physical interface are managed by the `ip-netns` management tool.

### The pause container creates the network namespace in the pod

When you create a pod, the container runtime first creates a network namespace for the containers. Each pod in the cluster has an additional hidden container running in the background called the pause container, responsible for creating and holding the network namespace.

For example, listing the pause containers running on a node reveals:

```bash
docker ps | grep pause
fa9666c1d9c6   registry.k8s.io/pause:3.4.1  "/pause"  k8s_POD_kube-dns-599484b884-sv2js…
44218e010aeb   registry.k8s.io/pause:3.4.1  "/pause"  k8s_POD_blackbox-exporter-55c457d…
5fb4b5942c66   registry.k8s.io/pause:3.4.1  "/pause"  k8s_POD_kube-dns-599484b884-cq99x…
8007db79dcf2   registry.k8s.io/pause:3.4.1  "/pause"  k8s_POD_konnectivity-agent-84f87c…
```

The pause container creates the network namespace with minimal code and goes to sleep, ensuring robust network namespace creation. If one of the containers in the pod crashes, the remaining container can still reply to network requests.

### The pod is assigned a single IP address

Inside the pod network namespace, an interface is created, and an IP address is assigned. This IP is shared between all containers within the pod.

To find the pod's IP address:

```bash
kubectl get pod multi-container-pod -o jsonpath={.status.podIP}
```

You can also verify the network namespace and interfaces from within the cluster node:

```bash
ip netns exec <namespace> ip a
```

### Inspecting pod to pod traffic in the cluster

When inspecting pod-to-pod traffic, there are two scenarios:

1. Traffic destined for a pod on the same node.
2. Traffic destined for a pod on a different node.

For a pod to communicate with other pods, it must first have access to the node's root namespace. This is achieved using a virtual ethernet pair (veth), connecting the pod namespace to the root namespace. Each newly created pod on the node will have a veth pair.

### The pod network namespace is connected to an ethernet bridge

An ethernet bridge at layer 2 of the OSI networking model connects each end of the virtual interfaces in the root namespace, allowing traffic to flow between virtual pairs.

### Tracing pod to pod traffic on the same node

For pods on the same node, Pod-A sends a packet to its default interface `eth0`, tied to one end of the veth pair, forwarding packets to the root namespace on the node.

### Tracing pod to pod communication on different nodes

For pods communicating across different nodes, an additional hop is required. The source node performs a bitwise operation to determine if the destination IP is on a different network. If so, the packet is forwarded to the default gateway of the node.

### The Container Network Interface - CNI

The Container Network Interface (CNI) handles networking within the current node. The kubelet interacts with the CNI plugins to ensure the network setup for pods.

### Inspecting pod to service traffic

To be detailed in subsequent sections.

### Intercepting and rewriting traffic with Netfilter and Iptables

To be detailed in subsequent sections.

### Inspecting responses from services

To be detailed in subsequent sections.
