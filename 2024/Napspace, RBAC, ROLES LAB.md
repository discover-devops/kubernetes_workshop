To set up and test the working of the Kubernetes Service Accounts (SA), Roles, RoleBindings, ClusterRole, ClusterRoleBindings, and deployments, follow these steps:

### Step-by-Step Guide to Setup and Test

#### 1. Apply the Namespace Configurations

Create the `test` and `prod` namespaces.

**namespace-test.yaml**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test
```

**namespace-prod.yaml**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
```

Apply the namespaces:

```sh
kubectl apply -f namespace-test.yaml
kubectl apply -f namespace-prod.yaml
```

#### 2. Create Service Accounts

Create service accounts for both namespaces.

**serviceaccount-test.yaml**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test-service-account
  namespace: test
```

**serviceaccount-prod.yaml**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prod-service-account
  namespace: prod
```

Apply the service accounts:

```sh
kubectl apply -f serviceaccount-test.yaml
kubectl apply -f serviceaccount-prod.yaml
```

#### 3. Create Roles

Create roles for the `test` and `prod` namespaces.

**role-test.yaml**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: test
  name: test-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "create", "delete"]
```

**role-prod.yaml**

```yaml
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

Apply the roles:

```sh
kubectl apply -f role-test.yaml
kubectl apply -f role-prod.yaml
```

#### 4. Create RoleBindings

Bind the roles to the service accounts in each namespace.

**rolebinding-test.yaml**

```yaml
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
```

**rolebinding-prod.yaml**

```yaml
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

Apply the role bindings:

```sh
kubectl apply -f rolebinding-test.yaml
kubectl apply -f rolebinding-prod.yaml
```

#### 5. Create ClusterRole

Create a cluster-wide role.

**clusterrole.yaml**

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

Apply the cluster role:

```sh
kubectl apply -f clusterrole.yaml
```

#### 6. Create ClusterRoleBinding

Bind the cluster role to both service accounts.

**clusterrolebinding.yaml**

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

Apply the cluster role binding:

```sh
kubectl apply -f clusterrolebinding.yaml
```

#### 7. Create Deployments

Deploy applications in each namespace.

**deployment-test.yaml**

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

**deployment-prod.yaml**

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

Apply the deployments:

```sh
kubectl apply -f deployment-test.yaml
kubectl apply -f deployment-prod.yaml
```

### Test the Access and Deployment

To test access as the Service Account, you can use the `kubectl run` command with the `--as` flag or create a pod that uses the service account.

#### Using `kubectl run` with `--as`

This method is straightforward for testing.

```sh
kubectl auth can-i create pods --as=system:serviceaccount:test:test-service-account -n test
kubectl auth can-i delete pods --as=system:serviceaccount:prod:prod-service-account -n prod
```

#### Creating a Pod to Test the Service Account

1. **Create a test pod spec file**:

**test-pod.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: test
spec:
  serviceAccountName: test-service-account
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
```

2. **Create the pod**:

```sh
kubectl apply -f test-pod.yaml
```

3. **Exec into the pod to test permissions**:

```sh
kubectl exec -it test-pod -n test -- sh
```

4. **Inside the pod, try performing actions to verify access**:

```sh
# List pods (should work)
kubectl get pods -n test

# Create a pod (should work)
kubectl run nginx --image=nginx -n test

# Delete a pod (should work)
kubectl delete pod nginx -n test
```

By following these steps, you can set up and test the Kubernetes Service Accounts, roles, role bindings, cluster roles, and cluster role bindings, ensuring that your configurations work as expected.
