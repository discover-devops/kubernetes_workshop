# Kubernetes Probes — Hands-On Labs

**Build Automate Architect**
*Context → Concept → Lab*

Two self-contained labs, run in order:

- **Lab 3A** — Readiness Probe: prevent traffic until the application is ready
- **Lab 3B** — Liveness Probe: automatically restart an unhealthy application

Each lab includes the full YAML, every command, the expected output, and an explanation of what's happening at each step. Copy and paste everything exactly as shown.

---

# Lab 3A — Readiness Probe: Prevent Traffic Until the Application Is Ready

## Objective

Understand how a **Readiness Probe** prevents a Kubernetes Service from sending traffic to a Pod until the application inside it is fully initialized.

## Learning Outcomes

By the end of this lab, you will understand:

- The difference between **Running** and **Ready**
- How a Readiness Probe works
- Why a Service only routes traffic to Ready Pods
- Why a Readiness Probe does **not** restart a Pod

---

## Architecture

```text
                    kubectl apply
                           │
                           ▼
                    Deployment Created
                           │
                           ▼
                      Pod Starts
                           │
                           ▼
                 Application Booting
                           │
                           ▼
                Readiness Probe Fails
                           │
                           ▼
                     READY = 0/1
                           │
                 Service ignores Pod
                           │
                           ▼
             Application becomes Ready
                           │
                           ▼
              Readiness Probe Succeeds
                           │
                           ▼
                     READY = 1/1
                           │
                           ▼
            Service starts routing traffic
```

---

## YAML File

**File name:** `03-probes-readiness.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-demo
  namespace: k8s-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: readiness-demo
  template:
    metadata:
      labels:
        app: readiness-demo
    spec:
      containers:
      - name: web
        image: nginxdemos/hello

        ports:
        - containerPort: 80

        # Simulate slow application startup
        command:
        - /bin/sh
        - -c
        - |
          sleep 30
          nginx -g 'daemon off;'

        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 3

---
apiVersion: v1
kind: Service
metadata:
  name: readiness-svc
  namespace: k8s-lab
spec:
  selector:
    app: readiness-demo

  ports:
  - port: 80
    targetPort: 80
```

> Make sure the `k8s-lab` namespace already exists in your cluster before applying this file. If it doesn't, create it with `kubectl create namespace k8s-lab`.

---

## Step 1 — Deploy the Application

```bash
kubectl apply -f 03-probes-readiness.yaml
```

**Expected output:**

```text
deployment.apps/readiness-demo created
service/readiness-svc created
```

**What's happening:** This creates a Deployment, a ReplicaSet, a Pod, and a Service. The application inside the container intentionally waits 30 seconds before starting, so we can clearly observe the "Running but Not Ready" state.

---

## Step 2 — Watch the Rollout

```bash
kubectl rollout status deployment/readiness-demo -n k8s-lab
```

**Expected output:**

```text
Waiting for deployment "readiness-demo" rollout to finish...
```

**What's happening:** The Deployment is waiting because the Pod has not yet passed its Readiness Probe. Kubernetes won't consider this rollout complete until the Pod reports Ready.

---

## Step 3 — Watch Pod Status

```bash
kubectl get pods -n k8s-lab -w
```

**Expected output (initially):**

```text
NAME                              READY   STATUS    RESTARTS   AGE
readiness-demo-xxxxxxx            0/1     Running   0          8s
```

**What to notice:**

```text
STATUS = Running
READY  = 0/1
```

The container has started successfully, but Kubernetes is **not yet allowing user traffic** to reach it. This is the most important observation in this lab.

---

## Step 4 — Describe the Pod

```bash
kubectl describe pod -l app=readiness-demo -n k8s-lab
```

Scroll to the **Events** section.

**Expected output:**

```text
Warning  Unhealthy

Readiness probe failed
connection refused
```

or

```text
HTTP probe failed
```

**What's happening:** The application is still starting, so it isn't responding on port 80 yet. Kubernetes keeps the Pod in the **Not Ready** state. No restart happens — this is Readiness, not Liveness.

---

## Step 5 — Check Service Endpoints

```bash
kubectl get endpoints readiness-svc -n k8s-lab
```

**Expected output (initially):**

```text
NAME             ENDPOINTS
readiness-svc    <none>
```

**What's happening:** The Pod exists, but the Service has no backend endpoints, because the Pod is not Ready. No client traffic can reach it yet.

---

## Step 6 — Wait for Startup

Wait approximately **30 seconds** for the application to finish initializing.

---

## Step 7 — Verify the Pod Again

```bash
kubectl get pods -n k8s-lab
```

**Expected output:**

```text
NAME                              READY   STATUS    RESTARTS   AGE
readiness-demo-xxxxxxx            1/1     Running   0          40s
```

**What's happening:** The Readiness Probe now succeeds. The Pod's status changes from `READY 0/1` to `READY 1/1` — it's now eligible to receive traffic.

---

## Step 8 — Verify Endpoints Again

```bash
kubectl get endpoints readiness-svc -n k8s-lab
```

**Expected output:**

```text
NAME             ENDPOINTS
readiness-svc    10.244.0.10:80
```

**What's happening:** Compare this to Step 5 — the endpoint went from `<none>` to a real Pod IP. The Service has now discovered a healthy backend and can route traffic to it.

---

## Step 9 — Test the Service

Open a port-forward:

```bash
kubectl port-forward svc/readiness-svc 8080:80 -n k8s-lab
```

In a **second terminal**, run:

```bash
curl http://localhost:8080
```

**Expected output:**

```text
Hello from NGINX Demo
```

---

## Key Observations

| Check | Before Ready | After Ready |
|---|---|---|
| Pod Status | Running | Running |
| READY Column | 0/1 | 1/1 |
| Restart Count | 0 | 0 |
| Service Endpoints | `<none>` | Pod IP |
| Traffic | No | Yes |

## Key Learnings

- ✅ A Pod can be **Running** but **Not Ready**.
- ✅ Readiness Probe controls **traffic**, not the Pod's lifecycle.
- ✅ A failed Readiness Probe **does not restart** the container.
- ✅ Kubernetes Services only route traffic to **Ready** Pods.

## Check Your Understanding

1. If the Pod is `Running`, why is `READY` still `0/1`?
2. Why does the Service show `<none>` for endpoints early on?
3. Did Kubernetes restart the Pod when the Readiness Probe failed? Why not?
4. What actually changed between `READY 0/1` and `READY 1/1`?

## Cleanup

```bash
kubectl delete -f 03-probes-readiness.yaml
```

---

# Lab 3B — Liveness Probe: Automatically Restart an Unhealthy Application

## Objective

Understand how a **Liveness Probe** continuously monitors application health and automatically restarts a container when the application becomes unhealthy — even while the container process is technically still running.

## Learning Outcomes

By the end of this lab, you will understand:

- How a Liveness Probe works
- The difference between Readiness and Liveness
- Why Kubernetes restarts unhealthy containers
- How to observe restart count
- How to inspect logs from a container's previous run

---

## Architecture

```text
                  kubectl apply
                         │
                         ▼
                   Deployment Created
                         │
                         ▼
                  Container Starts
                         │
                         ▼
                Application Healthy
                         │
                  Liveness Probe ✓
                         │
                         ▼
           Application becomes Unhealthy
                         │
                  Liveness Probe ✗
                         │
                         ▼
            Kubernetes kills Container
                         │
                         ▼
              New Container Starts
```

---

## YAML File

**File name:** `04-probes-liveness-bad.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: liveness-bad
  namespace: k8s-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: liveness-bad

  template:
    metadata:
      labels:
        app: liveness-bad

    spec:
      containers:
      - name: app
        image: busybox:1.36

        command:
        - /bin/sh
        - -c
        - |
          echo "Starting application..."

          touch /tmp/healthy

          # Simulate application becoming unhealthy
          (
            sleep 25
            echo "Application became unhealthy"
            rm -f /tmp/healthy
          ) &

          while true
          do
            echo "Application is running..."
            sleep 5
          done

        livenessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy

          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 2
```

---

## How This Demo Works

Initially, the file `/tmp/healthy` exists. Every few seconds the Liveness Probe runs:

```bash
cat /tmp/healthy
```

This succeeds — the file is there.

After **25 seconds**, the background script runs:

```bash
rm /tmp/healthy
```

Now the same `cat /tmp/healthy` command fails. Kubernetes concludes the application is unhealthy and restarts the container.

---

## Step 1 — Deploy the Application

```bash
kubectl apply -f 04-probes-liveness-bad.yaml
```

**Expected output:**

```text
deployment.apps/liveness-bad created
```

**What's happening:** The application starts normally, and the Liveness Probe initially succeeds.

---

## Step 2 — Watch the Pod

```bash
kubectl get pods -n k8s-lab -w
```

**Expected output (initially):**

```text
NAME                          READY   STATUS    RESTARTS   AGE
liveness-bad-xxxxx            1/1     Running   0          5s
```

Wait approximately **35 seconds**. You should then see:

```text
NAME                          READY   STATUS    RESTARTS   AGE
liveness-bad-xxxxx            1/1     Running   1          40s
```

Keep watching — after another cycle, `RESTARTS` becomes `2`, then `3`.

**What's happening:** The Pod name never changes. Only the container inside the Pod is restarted. This is Kubernetes' **self-healing** capability in action.

---

## Step 3 — Inspect Events

```bash
kubectl describe pod -l app=liveness-bad -n k8s-lab
```

Scroll to **Events**.

**Expected output:**

```text
Warning  Unhealthy
Liveness probe failed

Warning  Killing
Container failed liveness probe,
will be restarted

Normal  Started
Started container
```

**What's happening:** Notice the sequence — Liveness Failed → Container Killed → Container Started Again.

---

## Step 4 — Check the Restart Count

```bash
kubectl get pod -l app=liveness-bad \
-n k8s-lab \
-o jsonpath='{.items[0].status.containerStatuses[0].restartCount}'
```

**Expected output:**

```text
3
```

(or `4`, depending on timing)

**What's happening:** Every failed Liveness Probe increases the restart count by one.

---

## Step 5 — View Current Logs

```bash
kubectl logs -l app=liveness-bad -n k8s-lab --tail=20
```

**Expected output:**

```text
Starting application...
Application is running...
Application is running...
```

**What's happening:** These are logs from the **current** container. Since the container has restarted, the earlier logs (from before the restart) aren't part of this instance.

---

## Step 6 — View Previous Container Logs

```bash
kubectl logs \
-l app=liveness-bad \
-n k8s-lab \
--previous
```

**Expected output:**

```text
Starting application...
Application is running...
Application became unhealthy
```

**What's happening:** The `--previous` flag retrieves logs from the container instance that existed **before** the restart. This is extremely useful when debugging applications that crash or fail health checks.

---

## Step 7 — Watch Continuous Self-Healing

```bash
kubectl get pods -n k8s-lab -w
```

Observe the pattern:

```text
READY = 1/1
STATUS = Running
RESTARTS: 0 → 1 → 2 → 3 ...
```

The application keeps failing its health check on the same 25-second cycle, and Kubernetes keeps recovering it automatically.

---

## Key Observations

| Observation | Meaning |
|---|---|
| Pod remains Running | Kubernetes keeps the Pod alive |
| Restart Count increases | Container is being restarted |
| Pod name doesn't change | Same Pod, new container instance |
| Current logs reset | New container starts fresh |
| Previous logs available | Shows logs from before the restart |
| Events show "Liveness probe failed" | Health check failed |
| Events show "Killing" | Kubernetes restarted the container |

## Readiness vs. Liveness — Side by Side

| Readiness Probe | Liveness Probe |
|---|---|
| Controls traffic | Controls restarts |
| Pod remains running | Container is restarted |
| Service removes Pod from endpoints | Kubernetes kills the container |
| Restart Count stays at 0 | Restart Count increases |

## Key Learnings

- ✅ A Liveness Probe continuously checks application health.
- ✅ If the application becomes unhealthy, Kubernetes automatically restarts the container.
- ✅ The Pod itself is **not** recreated — only the container inside it is restarted.
- ✅ Restart count is a quick signal of recurring application failures.
- ✅ `kubectl logs --previous` is invaluable for diagnosing failures that happened before the most recent restart.

## Check Your Understanding

1. Did Kubernetes create a new Pod, or just restart the container?
2. Why does the Pod name stay the same while the restart count increases?
3. Why are the current logs different from the previous logs?
4. If this were a production Java application with a deadlock, how would a Liveness Probe help?

## Cleanup

```bash
kubectl delete -f 04-probes-liveness-bad.yaml
```
