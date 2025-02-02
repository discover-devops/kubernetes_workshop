Role-Based Access Control (RBAC) is a crucial system for managing access within a Kubernetes cluster, especially when multiple team members require varying levels of security from each other. RBAC in Kubernetes serves to restrict and define access permissions within the cluster.

In general, RBAC operates by associating roles with specific verbs (e.g., get, list) and nouns (e.g., pods, volumes), essentially determining what actions can be performed on Kubernetes objects. Kubernetes offers predefined roles, and users can also create custom roles using YAML files. However, it's essential to note that a role alone doesn't grant any permissions. To enable access, a role must be bound to a specific user or entity through role bindings.

Role bindings establish the connection between a role and the entity authorized to execute API calls, effectively mapping permissions. Users can have multiple roles, and often, roles are associated with groups via role bindings.

It's important to distinguish between two types of resources in Kubernetes:

1. **Namespace-based resources:** These resources are confined within a specific namespace, serving as the scope for resource control.

2. **Cluster-level resources:** These resources operate at the overall cluster level.

There are two primary types of role bindings:

1. **Cluster Role Bindings:** These provide access at the cluster level, affecting resources across all namespaces.

2. **Role Bindings:** These grant access at the namespace level, influencing resources within a specific namespace.

In summary, RBAC in Kubernetes revolves around defining roles that specify what actions can be taken on resources, and these roles are then bound to entities through role bindings to grant the necessary permissions.





---

### **Step 1: Setup Initial Resources**

Ensure you have already applied the following resources:
1. Namespaces (`test` and `prod`).
2. Service accounts (`test-service-account` and `prod-service-account`).
3. Roles and RoleBindings for both namespaces.

Apply these commands if not done already:
```bash
kubectl apply -f namespace-test.yaml
kubectl apply -f namespace-prod.yaml
kubectl apply -f serviceaccount-test.yaml
kubectl apply -f serviceaccount-prod.yaml
kubectl apply -f role-test.yaml
kubectl apply -f role-prod.yaml
kubectl apply -f rolebinding-test.yaml
kubectl apply -f rolebinding-prod.yaml
```

---

### **Step 2: Verify Default Access Without ClusterRoleBinding**

1. **Create a test pod in the `test` namespace to verify access for the `test-service-account`:**

   Create a YAML file (`test-pod.yaml`) to simulate access:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: test-access-check
     namespace: test
   spec:
     containers:
     - name: access-check
       image: bitnami/kubectl:latest
       command: ["sleep", "3600"]
     serviceAccountName: test-service-account
   ```
   Apply the YAML:
   ```bash
   kubectl apply -f test-pod.yaml
   ```

2. **Execute the pod to check access permissions:**
   - Open a shell inside the pod:
     ```bash
     kubectl exec -it test-access-check -n test -- /bin/sh
     ```

   - Test if the pod can list resources within the `test` namespace:
     ```sh
     kubectl get pods -n test
     ```
     The command should succeed because the **test-role** grants permissions in the `test` namespace.

   - Test access to resources in the `prod` namespace:
     ```sh
     kubectl get pods -n prod
     ```
     This should return an error like:
     ```sh
     Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:test:test-service-account" cannot list resource "pods" in API group "" in the namespace "prod"
     ```

   - Exit the shell:
     ```sh
     exit
     ```

---

### **Step 3: Repeat the Verification for the Prod Namespace**

1. **Create a pod for access verification in the `prod` namespace**:
   Create a YAML file (`prod-pod.yaml`):
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: prod-access-check
     namespace: prod
   spec:
     containers:
     - name: access-check
       image: bitnami/kubectl:latest
       command: ["sleep", "3600"]
     serviceAccountName: prod-service-account
   ```
   Apply the YAML:
   ```bash
   kubectl apply -f prod-pod.yaml
   ```

2. **Execute the pod to check access permissions**:
   - Open a shell inside the pod:
     ```bash
     kubectl exec -it prod-access-check -n prod -- /bin/sh
     ```

   - Test access to resources in the `prod` namespace:
     ```sh
     kubectl get pods -n prod
     ```
     The command should succeed because the **prod-role** grants permissions in the `prod` namespace.

   - Test access to resources in the `test` namespace:
     ```sh
     kubectl get pods -n test
     ```
     This should return an error like:
     ```sh
     Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:prod:prod-service-account" cannot list resource "pods" in API group "" in the namespace "test"
     ```

   - Exit the shell:
     ```sh
     exit
     ```

---

### **Step 4: Verify the RBAC Configuration**

Run the following commands to confirm that RBAC is correctly applied:

1. **List Roles in the test namespace**:
   ```bash
   kubectl get roles -n test
   ```

2. **Describe the Role to check permissions**:
   ```bash
   kubectl describe role test-role -n test
   ```

3. **List RoleBindings in the test namespace**:
   ```bash
   kubectl get rolebindings -n test
   ```

4. **Describe the RoleBinding to confirm the association**:
   ```bash
   kubectl describe rolebinding test-rolebinding -n test
   ```

Repeat these steps for the `prod` namespace.

---

### **Step 5: Verify the Current Permissions of Each Service Account**

Run the following commands to check what each service account can do:

1. **Test the `test-service-account` in the `test` namespace**:
   ```bash
   kubectl auth can-i list pods --as=system:serviceaccount:test:test-service-account -n test
   ```

   Output should be:
   ```sh
   yes
   ```

2. **Test the `test-service-account` in the `prod` namespace**:
   ```bash
   kubectl auth can-i list pods --as=system:serviceaccount:test:test-service-account -n prod
   ```

   Output should be:
   ```sh
   no
   ```

3. **Test the `prod-service-account` in the `prod` namespace**:
   ```bash
   kubectl auth can-i list pods --as=system:serviceaccount:prod:prod-service-account -n prod
   ```

   Output should be:
   ```sh
   yes
   ```

4. **Test the `prod-service-account` in the `test` namespace**:
   ```bash
   kubectl auth can-i list pods --as=system:serviceaccount:prod:prod-service-account -n test
   ```

   Output should be:
   ```sh
   no
   ```

---

### **Step 6: Apply ClusterRole and ClusterRoleBinding**

Once you have verified that the service accounts have access only to their respective namespaces, apply the **ClusterRole** and **ClusterRoleBinding** to grant cluster-wide permissions:

```bash
kubectl apply -f clusterrole.yaml
kubectl apply -f clusterrolebinding.yaml
```

---

### **Step 7: Re-Test Cluster-Wide Access**

1. Re-run the access tests from Step 2 and Step 3.  
   Now, both `test-service-account` and `prod-service-account` should have access to resources in all namespaces due to the **ClusterRoleBinding**.

2. Test access with the `auth can-i` command:
   ```bash
   kubectl auth can-i list pods --as=system:serviceaccount:test:test-service-account -n prod
   ```

   Output should now be:
   ```sh
   yes
   ```

---

### **Summary**

- Before applying the **ClusterRole** and **ClusterRoleBinding**, each service account has access only to its respective namespace.
- After applying the **ClusterRoleBinding**, the service accounts gain cluster-wide permissions.
- You can verify permissions using:
  - The `kubectl exec` command inside a pod.
  - The `kubectl auth can-i` command.  
   
This ensures that RBAC permissions are properly implemented and verified.
