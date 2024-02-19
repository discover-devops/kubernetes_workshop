
# Labels and Annotations

## Metadata in Kubernetes

Metadata is essential for managing resources in a cluster, especially when dealing with potentially thousands of resources. Labels and annotations are two concepts used to add metadata to Kubernetes objects, such as pods.

## Labels

Labels are key-value pairs that serve as metadata for Kubernetes objects. They can be attached to objects like pods at creation or modified during runtime. Each key must be unique for an object.

### Example of Labels in YAML:

```yaml
metadata:
  labels:
    key1: value1
    key2: value2
```

### Why Labels?

Labels are used for organizing objects and filtering subsets based on those labels. They are helpful for running specific pods on selected nodes. Use cases include organizing objects based on teams or organizations within a company.

### Creating a Pod with Labels:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  labels:
    app: nginx
    team: team_a
spec:
  containers:
  - name: container1
    image: nginx
```

Create the pod with:

```bash
kubectl create -f pod.yaml
kubectl get pod pod1
kubectl describe pod pod1
```

### Adding Labels to a Running Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod2
spec:
  containers:
  - name: container2
    image: nginx
```

Create the pod with:

```bash
kubectl create -f pod2.yaml
kubectl get pod pod2
kubectl describe pod pod2.yaml
kubectl label pod pod2 app=prod
kubectl describe pod pod2
kubectl label pod pod2 team=team_A type=test
kubectl describe pod pod2
```

### Modifying and Deleting Labels for a Running Pod:

Modify the label:

```bash
kubectl label --overwrite pod pod2 app=nginx-application
kubectl describe pod pod2
```

Delete the label:

```bash
kubectl label pod pod2 app-
kubectl describe pod pod2
```

### Selecting Kubernetes Objects Using Label Selectors:

Use label selectors to group objects:

```bash
kubectl get pods -l {label_selector}
```

Examples:

```bash
kubectl get pods -l environment=prod
kubectl get pods -l team!=devops
kubectl get pods -l environment=prod,team!=devops
```

### Selecting Pods Using Equality-Based Label Selectors:

Examples in YAML:

```yaml
# pod3.yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-production
  labels:
    environment: production
    role: frontend
spec:
  containers:
  - name: application-container
    image: nginx
```

```yaml
# pod4.yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-production
  labels:
    environment: production
    role: backend
spec:
  containers:
  - name: application-container
    image: nginx
```

```yaml
# pod5.yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-staging
  labels:
    environment: staging
    role: frontend
spec:
  containers:
  - name: application-container
    image: nginx
```

Create pods:

```bash
kubectl create -f pod3.yaml
kubectl create -f pod4.yaml
kubectl create -f pod5.yaml
kubectl get pod backend-production --show-labels
kubectl get pod frontend-staging --show-labels
kubectl get pods -l environment=production
kubectl get pods -l role=frontend,environment=staging
```

## Annotations

Labels have constraints on values, such as character limits and alphanumeric requirements. Annotations have fewer constraints and can store unstructured information related to Kubernetes objects.

