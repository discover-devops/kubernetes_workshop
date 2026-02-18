###  What is a Rolling Update in Kubernetes?

A **Rolling Update** is a deployment strategy where **pods are gradually updated** to the new version, **without downtime**. It replaces old pods with new ones **incrementally**, ensuring a minimum number of pods are always available during the update.

---

###  Benefits of Rolling Updates

* Zero downtime
* Controlled release
* Can pause or rollback if needed

---

###  Deployment Upgrade Strategies in Kubernetes

Kubernetes supports **two main update strategies**:

| Strategy                    | Description                                                        |
| --------------------------- | ------------------------------------------------------------------ |
| **RollingUpdate** (default) | Gradually replaces pods with new version                           |
| **Recreate**                | Terminates all old pods before creating new ones (causes downtime) |

You specify this in the `strategy` field of the Deployment YAML.

```yaml
strategy:
  type: RollingUpdate  # or Recreate
```

---

###  Step-by-Step: Upgrade a Deployment using Rolling Update

Let’s say you have a Deployment running `nginx:1.21` and you want to update it to `nginx:1.25`.

---

####  Step 1: View the current deployment

```bash
kubectl get deployments
kubectl describe deployment nginx-deployment
```

---

####  Step 2: Edit or update the deployment YAML

Option A: Use `kubectl set image` to update image directly:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.25
```

Option B: Manually update the YAML:

```yaml
# nginx-deployment.yaml
spec:
  template:
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
```

Then apply:

```bash
kubectl apply -f nginx-deployment.yaml
```

---

####  Step 3: Monitor the rolling update progress

```bash
kubectl rollout status deployment/nginx-deployment
```

It will show something like:

```
deployment "nginx-deployment" successfully rolled out
```

---

####  Step 4: Verify the pods

```bash
kubectl get pods -l app=nginx
```

Check that all pods are using the new version.

---

###  Optional: Rollback to Previous Version

If the update caused problems:

```bash
kubectl rollout undo deployment/nginx-deployment
```

This reverts the deployment to the previous working version.

---

###  Bonus: Control Rolling Update Behavior

You can customize these parameters in the deployment YAML:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1           # max extra pods (above desired replicas) during update
    maxUnavailable: 1     # max pods allowed to be unavailable during update
```

---


