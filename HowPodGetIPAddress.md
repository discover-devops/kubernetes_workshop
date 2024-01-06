## How an IP Address Gets Assigned to a Pod in Kubernetes

### Container Runtime Initialization:

- When a Pod is created, the container runtime (e.g., Docker, containerd) starts the containers within the Pod.
- Containers within a Pod share the same network namespace, using the same IP address and port space.

### Container Network Interface (CNI) Plugins:

- Kubernetes uses the Container Network Interface (CNI) to configure networking interfaces for Linux containers.
- The CNI plugin is chosen based on configuration and may be specific to a cloud platform or a standalone Kubernetes cluster.
- IP address assignment and networking within the Pod are managed by the selected CNI plugin.

### CNI Configuration and IP Address Management (IPAM):

- Kubernetes delegates IP address assignment to the CNI plugin specified in the CNI configuration.
- IP Address Management (IPAM) varies based on the chosen CNI plugin and IPAM method.

### Pod Creation Trigger:

- When a Pod is created, the kubelet on the node sends a request to the configured CNI plugin to set up the network for the Pod.

### IP Address Assignment:

- The CNI plugin, upon receiving the request, interacts with the IPAM to obtain an available IP address for the new Pod.
- IPAM ensures the assigned IP address is unique within the network.
- Once an IP address is obtained, the CNI plugin configures the network namespace for the Pod, applying the assigned IP address and setting up necessary routes.

### Network Setup and Storage:

- The CNI plugin establishes the network configuration within the Pod's namespace, ensuring proper connectivity.
- Details of the assigned IP address, routes, and network setup are stored by the CNI plugin for future reference and cleanup.

### Dynamic IP Address Management:

- IP addresses may change during Pod restarts, rescheduling, or scaling operations.
- The CNI plugin and IPAM handle dynamic scenarios by reassigning IP addresses, ensuring proper network functioning.
