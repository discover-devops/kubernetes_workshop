### Kubernetes Namespace

A Kubernetes Namespace is a way to divide cluster resources between multiple users. Namespaces are intended for use in environments with many users spread across multiple teams, or projects. Namespaces provide a mechanism for isolating groups of resources within a single cluster.

### YAML to Create Namespaces

1. **Create Namespaces `test` and `prod`**

```yaml
# test namespace
apiVersion: v1
kind: Namespace
metadata:
  name: test

# prod namespace
apiVersion: v1
kind: Namespace
metadata:
  name: prod
```

### Create Service Account, Role, RoleBinding, ClusterRole, and ClusterRoleBinding for the Namespaces

1. **Service Account**

```yaml
# Service Account for test namespace
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test-service-account
  namespace: test

# Service Account for prod namespace
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prod-service-account
  namespace: prod
```

2. **Role**

```yaml
# Role for test namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: test
  name: test-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "create", "delete"]

# Role for prod namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: prod
  name: prod-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "create", "delete"]
```

3. **RoleBinding**

```yaml
# RoleBinding for test namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: test-rolebinding
  namespace: test
subjects:
- kind: ServiceAccount
  name: test-service-account
  namespace: test
roleRef:
  kind: Role
  name: test-role
  apiGroup: rbac.authorization.k8s.io

# RoleBinding for prod namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: prod-rolebinding
  namespace: prod
subjects:
- kind: ServiceAccount
  name: prod-service-account
  namespace: prod
roleRef:
  kind: Role
  name: prod-role
  apiGroup: rbac.authorization.k8s.io
```

4. **ClusterRole**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "create", "delete"]
```

5. **ClusterRoleBinding**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-rolebinding
subjects:
- kind: ServiceAccount
  name: test-service-account
  namespace: test
- kind: ServiceAccount
  name: prod-service-account
  namespace: prod
roleRef:
  kind: ClusterRole
  name: cluster-role
  apiGroup: rbac.authorization.k8s.io
```

### Deployment for Each Namespace

1. **Deployment in `test` Namespace**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
  namespace: test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      serviceAccountName: test-service-account
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

2. **Deployment in `prod` Namespace**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-deployment
  namespace: prod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: prod-app
  template:
    metadata:
      labels:
        app: prod-app
    spec:
      serviceAccountName: prod-service-account
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

### Applying the YAML files

To apply these YAML configurations, save each snippet to a file and then use the `kubectl apply` command:

```sh
kubectl apply -f namespace-test.yaml
kubectl apply -f namespace-prod.yaml
kubectl apply -f serviceaccount-test.yaml
kubectl apply -f serviceaccount-prod.yaml
kubectl apply -f role-test.yaml
kubectl apply -f role-prod.yaml
kubectl apply -f rolebinding-test.yaml
kubectl apply -f rolebinding-prod.yaml
kubectl apply -f clusterrole.yaml
kubectl apply -f clusterrolebinding.yaml
kubectl apply -f deployment-test.yaml
kubectl apply -f deployment-prod.yaml
```

### Example of Roles and ClusterRoles in Action

#### Roles in Namespaces
- A Role (e.g., `test-role` or `prod-role`) defines what actions can be performed within a specific namespace. 
- The RoleBinding binds this Role to a Service Account, ensuring the Service Account has the necessary permissions within that namespace.
  
Example: The `test-service-account` in the `test` namespace can `get`, `list`, `create`, and `delete` pods due to its `RoleBinding` to `test-role`.

#### ClusterRoles Across Namespaces
- A ClusterRole provides similar permissions but across all namespaces or at the cluster level.
- The ClusterRoleBinding binds the ClusterRole to a Service Account, granting it permissions cluster-wide.

Example: The `test-service-account` and `prod-service-account` can perform cluster-wide actions like `get`, `list`, `create`, and `delete` pods due to their `ClusterRoleBinding` to `cluster-role`.

By following this step-by-step approach, you can create namespaces, service accounts, roles, role bindings, cluster roles, and cluster role bindings, and deploy applications with the appropriate permissions within a Kubernetes cluster.
