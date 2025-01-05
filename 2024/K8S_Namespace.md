### **What is a Kubernetes Namespace?**

A **Namespace** in Kubernetes is a virtual cluster within a physical cluster. It provides a way to divide cluster resources between multiple users or teams. Each Namespace acts as a logical boundary for resources and enables resource isolation and organization.


![image](https://github.com/user-attachments/assets/85e3150d-411e-4e12-8dcb-439dd169c809)


---

### **Key Features of Namespaces**
1. **Resource Isolation**:
   - Separate teams, projects, or applications can have their own resources without interference.
2. **Resource Quotas**:
   - Control resource usage (e.g., CPU, memory) within a Namespace.
3. **Organization**:
   - Helps group related resources for better management.
4. **Security**:
   - Role-Based Access Control (RBAC) can restrict user access to specific Namespaces.

---

### **Default Namespaces in Kubernetes**
1. **default**:
   - Used when no Namespace is specified.
2. **kube-system**:
   - Contains system components like kube-dns and the API server.
3. **kube-public**:
   - A Namespace that is readable by all users.
4. **kube-node-lease**:
   - Used for node heartbeats (since Kubernetes v1.13).

---

### **Use Cases for Namespaces**
1. **Environment Separation**:
   - Separate `dev`, `staging`, and `prod` environments.
2. **Team or Project Segmentation**:
   - Different teams or projects can operate in isolated Namespaces.
3. **Testing and Debugging**:
   - Run multiple versions of an application in isolated spaces.

---

### **Step-by-Step Lab: Working with Namespaces**

#### **Step 1: Create a Namespace**

1. **Create a Namespace YAML**:
   Save the following YAML as `namespace.yaml`:
   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: my-namespace
   ```

2. **Apply the YAML**:
   ```bash
   kubectl apply -f namespace.yaml
   ```

3. **Verify the Namespace**:
   ```bash
   kubectl get namespaces
   ```

---

#### **Step 2: Create Resources in the Namespace**

1. **Deploy an NGINX Pod in `my-namespace`**:
   Save the following YAML as `nginx-deployment.yaml`:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx
     namespace: my-namespace
   spec:
     replicas: 2
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

2. **Apply the YAML**:
   ```bash
   kubectl apply -f nginx-deployment.yaml
   ```

3. **Verify the Deployment**:
   ```bash
   kubectl get deployments -n my-namespace
   kubectl get pods -n my-namespace
   ```

---

#### **Step 3: Set a Default Namespace for Your `kubectl` Context**

1. **Check the Current Context**:
   ```bash
   kubectl config current-context
   ```

2. **Set `my-namespace` as the Default Namespace**:
   ```bash
   kubectl config set-context --current --namespace=my-namespace
   ```

3. **Test Default Namespace Behavior**:
   - Run a `kubectl` command without specifying the Namespace:
     ```bash
     kubectl get pods
     ```

   - It should show Pods in `my-namespace`.

---

#### **Step 4: Resource Quotas in a Namespace**

1. **Create a Resource Quota YAML**:
   Save the following YAML as `resource-quota.yaml`:
   ```yaml
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: my-quota
     namespace: my-namespace
   spec:
     hard:
       pods: "5"                  # Maximum number of Pods
       requests.cpu: "2"          # Total CPU requests
       requests.memory: "1Gi"     # Total memory requests
       limits.cpu: "4"            # Total CPU limits
       limits.memory: "2Gi"       # Total memory limits
   ```

2. **Apply the YAML**:
   ```bash
   kubectl apply -f resource-quota.yaml
   ```

3. **Verify the Quota**:
   ```bash
   kubectl get resourcequota -n my-namespace
   ```

4. **Test the Quota**:
   - Try deploying more than 5 Pods or exceeding the resource limits to observe the enforced restrictions.

---

#### **Step 5: Clean Up**

1. **Delete the Resources**:
   ```bash
   kubectl delete namespace my-namespace
   ```

---

### **Example Use Case: Isolating Dev and Prod Environments**

1. **Create Two Namespaces**:
   - `dev-environment`
   - `prod-environment`

2. **Deploy Different Versions of the Application**:
   - Use a `:dev` Docker image tag in `dev-environment`.
   - Use a `:stable` Docker image tag in `prod-environment`.

3. **Set Different Resource Quotas**:
   - Allocate fewer resources for `dev-environment`.
   - Allocate more resources for `prod-environment`.

4. **Test Application Behavior in Isolation**:
   - Developers can test in `dev-environment` without affecting the production system in `prod-environment`.

---

### **Commands Cheat Sheet**
| Command                                      | Description                                |
|----------------------------------------------|--------------------------------------------|
| `kubectl get namespaces`                     | List all Namespaces in the cluster.       |
| `kubectl apply -f <file>`                    | Apply a resource definition file.         |
| `kubectl config set-context --current --namespace=<namespace>` | Set default Namespace for the current context. |
| `kubectl delete namespace <namespace>`       | Delete a specific Namespace.              |
| `kubectl get pods -n <namespace>`            | List Pods in a specific Namespace.        |

---

### **Outcome**
1. You learned to create and manage Namespaces.
2. You deployed applications in isolated Namespaces.
3. You enforced resource quotas to control resource usage.

Namespaces are a vital part of Kubernetes for organizing and isolating workloads, making them especially useful in multi-tenant environments or when managing environments like `dev`, `staging`, and `prod`.
