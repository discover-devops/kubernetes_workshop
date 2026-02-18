**step-by-step document** to:

1.  Create a deployment with `nginx:1.21`
2.  Upgrade it to `nginx:1.25` using a **rolling update**

---

##  Step-by-Step Guide: Upgrade a Kubernetes Deployment (Nginx 1.21 → 1.25)

---

###  Step 1: Create a Deployment with nginx:1.21

####  File: `nginx-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
          ports:
            - containerPort: 80
```

####  Apply the Deployment:

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployments
kubectl get pods
```

---

###  Step 2: Verify the Version

```bash
kubectl describe deployment nginx-deployment | grep Image
```

 Output:

```
Image:  nginx:1.21
```

---

###  Step 3: Upgrade to nginx:1.25

#### Option 1: Use `kubectl set image`

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.25
```

#### Option 2: Update the YAML

Edit `nginx-deployment.yaml`:

```yaml
          image: nginx:1.25
```

Then apply:

```bash
kubectl apply -f nginx-deployment.yaml
```

---

###  Step 4: Monitor the Rolling Update

```bash
kubectl rollout status deployment/nginx-deployment
```

 Output:

```
deployment "nginx-deployment" successfully rolled out
```

---

###  Step 5: Confirm New Version

```bash
kubectl describe deployment nginx-deployment | grep Image
```

 Output:

```
Image:  nginx:1.25
```

Check pods:

```bash
kubectl get pods -o wide
```

---

###  Step 6 (Optional): Rollback to nginx:1.21

```bash
kubectl rollout undo deployment/nginx-deployment
```

---


