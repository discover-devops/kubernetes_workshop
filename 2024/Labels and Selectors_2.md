step-by-step tutorial on working with labels, selectors, and annotations in Kubernetes.

### Step 1: Creating a Pod with Labels

Create a YAML file named `mypod.yaml` with the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  labels:
    app: nginx
    env: production
spec:
  containers:
  - name: my-container
    image: nginx
```

Apply the YAML file to create the pod:

```bash
kubectl apply -f mypod.yaml
```

### Step 2: Adding Labels to a Running Pod

Create a YAML file named `mypod1.yaml` with the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod1
spec:
  containers:
  - name: my1-container
    image: nginx
```

Apply the YAML file to create the pod:

```bash
kubectl apply -f mypod1.yaml
```

Now, add labels to the running pod:

```bash
kubectl label pod mypod1 app=nginx
```

### Step 3: Modifying and Deleting Existing Labels

Create a YAML file named `pod3.yaml` with the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod3
  labels:
    app: nginx
spec:
  containers:
  - name: mt-container3
    image: nginx
```

Apply the YAML file to create the pod:

```bash
kubectl apply -f pod3.yaml
```

Now, overwrite the existing label:

```bash
kubectl label --overwrite pod pod3 app=nginx-demo
```

### Step 4: Selecting Kubernetes Objects Using Label Selectors

Create three pods with different labels:

- `pod-frontend-production.yaml`
- `pod-backend-production.yaml`
- `pod-frontend-staging.yaml`

Ensure each pod YAML contains appropriate labels.

```bash
kubectl create -f pod-frontend-production.yaml
kubectl create -f pod-backend-production.yaml
kubectl create -f pod-frontend-staging.yaml
```

Now, you can select pods based on label selectors:

```bash
kubectl get pods -l environment=production
kubectl get pods -l role=frontend,environment=staging
```

### Step 5: Working with Annotations

Create a pod YAML file with annotations:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-annotations
  annotations:
    commit-SHA: d6s9shb82365yg4ygd782889us28377gf6
    JIRA-issue: "https://your-jira-link.com/issue/ABC-1234"
    timestamp: "123456789"
    owner: "https://internal-link.to.website/username"
spec:
  containers:
  - name: application-container
    image: nginx
```

Apply the YAML file to create the pod:

```bash
kubectl apply -f pod-with-annotations.yaml
```

Now, you can view the annotations of the pod:

```bash
kubectl describe pod pod-with-annotations
```

That's it! You've created, labeled, selected, modified, and annotated Kubernetes pods using `kubectl` commands.
