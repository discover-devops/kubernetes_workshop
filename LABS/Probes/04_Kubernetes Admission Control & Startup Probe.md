# Kubernetes Admission Control & Startup Probe — Hands-On Labs

**Build Automate Architect**
*Context → Concept → Lab*

Two self-contained labs:

- **Lab 4** — Admission Controller: enforcing security policy with Pod Security Admission (PSA)
- **Lab 5** — Startup Probe: protecting slow-starting applications from premature restarts

Each lab includes the full YAML, every command, expected output, and an explanation of what's happening at each step. Copy and paste everything exactly as shown.

---

# Lab 4 — Admission Controller (Pod Security Admission)

## Objective

Learn how an **Admission Controller** can enforce security policies before Kubernetes ever stores an object in `etcd`.

## Learning Outcomes

By the end of this lab, you will understand:

- Authentication vs. Authorization vs. Admission Controller
- How Admission Controllers enforce cluster-wide policies
- What Pod Security Admission (PSA) is
- Why a Pod can be rejected even when the user has permission to create Pods

---

## Architecture

```text
                kubectl apply
                      │
                      ▼
               kube-apiserver
                      │
          Authentication (Who are you?)
                      │
          Authorization (Can you do it?)
                      │
      Admission Controller (Is this Pod allowed?)
               │                    │
               │                    │
            Reject               Accept
               │                    │
               ▼                    ▼
          Error Returned         Store in etcd
                                      │
                                      ▼
                                  Scheduler
                                      │
                                      ▼
                                   Worker Node
```

---

## Step 1 — Create a Restricted Namespace

We'll create a namespace with Pod Security Admission enabled.

**File:** `07-admission-policy.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: k8s-lab-restricted
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Apply it:**

```bash
kubectl apply -f 07-admission-policy.yaml
```

**Expected output:**

```text
namespace/k8s-lab-restricted created
```

**Verify the labels:**

```bash
kubectl get namespace k8s-lab-restricted --show-labels
```

**Expected output:**

```text
NAME                   STATUS   AGE   LABELS
k8s-lab-restricted     Active        pod-security.kubernetes.io/enforce=baseline,...
```

**What's happening:** The namespace now carries a built-in security policy. Any Pod created inside it must follow the **Baseline Pod Security Standard**.

---

## Step 2 — Create a Non-Compliant Pod

Now let's intentionally violate the security policy.

**File:** `05-admission-bad-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: k8s-lab-restricted

spec:
  containers:
  - name: bad-container
    image: nginx:stable

    securityContext:
      privileged: true
      runAsUser: 0
```

**Apply it:**

```bash
kubectl apply -f 05-admission-bad-pod.yaml
```

**Expected output:**

```text
Error from server (Forbidden):

pods "privileged-pod" is forbidden:

violates PodSecurity "baseline"

container "bad-container"
must not set securityContext.privileged=true
```

**What's happening:** Authentication succeeded. Authorization succeeded — you do have permission to create Pods. But the **Admission Controller rejected the request** because it violated the namespace's Pod Security policy. The request never reached `etcd`.

---

## Step 3 — Verify That No Pod Was Created

```bash
kubectl get pods -n k8s-lab-restricted
```

**Expected output:**

```text
No resources found.
```

**What's happening:** It's easy to assume Kubernetes created the Pod and then deleted it — that's not what happened. The Pod was **never created** in the first place. The Scheduler never even saw the request; it died inside the API Server, before `etcd`.

---

## Step 4 — Create a Compliant Pod

Now let's create a Pod that follows the policy.

**File:** `06-admission-good-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
  namespace: k8s-lab-restricted

spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000

    seccompProfile:
      type: RuntimeDefault

  containers:
  - name: app
    image: nginx:stable

    securityContext:
      allowPrivilegeEscalation: false

      capabilities:
        drop:
        - ALL
```

**Apply it:**

```bash
kubectl apply -f 06-admission-good-pod.yaml
```

**Expected output:**

```text
pod/compliant-pod created
```

**What's happening:** This Pod satisfies the Baseline security policy, so the Admission Controller allows the request. The object is stored in `etcd`, and the Scheduler assigns it to a worker node.

---

## Step 5 — Verify the Pod

```bash
kubectl get pods -n k8s-lab-restricted
```

**Expected output:**

```text
NAME              READY   STATUS    RESTARTS   AGE
compliant-pod     1/1     Running   0          15s
```

**What's happening:** The Pod passed all three checks — Authentication ✔, Authorization ✔, Admission Controller ✔.

---

## Complete Request Flow

```text
kubectl apply
      │
      ▼
API Server
      │
      ▼
Authentication
(Who are you?)
      │
      ▼
Authorization
(Do you have permission?)
      │
      ▼
Admission Controller
(Is this resource allowed?)
      │
 ┌────┴─────┐
 │          │
Reject     Accept
 │           │
 ▼           ▼
Error      etcd
             │
             ▼
        Scheduler
             │
             ▼
        Worker Node
```

---

## Key Observations

| Observation | Meaning |
|---|---|
| User has permission to create Pods | Authorization passed |
| Pod still rejected | Admission Controller enforced policy |
| `kubectl get pods` shows nothing | Pod was never persisted in etcd |
| Good Pod gets created | Policy requirements were satisfied |

## Key Learnings

- ✅ Authentication answers **"Who are you?"**
- ✅ Authorization answers **"Can you perform this action?"**
- ✅ Admission Controller answers **"Is this resource compliant with cluster policies?"**
- ✅ If the Admission Controller rejects the request, the object is **never stored in etcd**.

## Check Your Understanding

1. Before applying the bad Pod — do you think it will be created? Why?
2. After it's rejected — which component rejected it, and at what stage?
3. Why does `kubectl get pods` show nothing at all, rather than showing a deleted or failed Pod?

## Cleanup

```bash
kubectl delete namespace k8s-lab-restricted
```

---

# Lab 5 — Startup Probe: Protect Slow-Starting Applications

## Objective

Learn how a **Startup Probe** allows a slow-starting application to fully initialize before Kubernetes begins running Liveness Probes against it.

## Learning Outcomes

By the end of this lab, you will understand:

- Why the Startup Probe exists
- The difference between Startup and Liveness Probes
- How the Startup Probe prevents unnecessary restarts
- The order in which Startup and Liveness Probes execute

---

## Real-World Context

Suppose you have a Spring Boot application whose startup activities include:

- Loading 500 classes
- Connecting to MySQL
- Connecting to Redis
- Loading cache
- Initializing a thread pool

This takes about **40 seconds**.

If a Liveness Probe starts checking after only 10 seconds:

```text
Container Starts
      │
10 sec
      │
Liveness Probe
      │
Application still starting
      │
Probe Fails
      │
Container Restarted
```

The application never gets the chance to finish starting — it gets killed mid-boot, every time.

**The Startup Probe solves this:**

```text
Container Starts
      │
Startup Probe
      │
Application Starting...
      │
Application Starting...
      │
Application Ready
      │
Startup Probe Success
      │
Liveness Probe Begins
```

---

## Step 1 — Create the Namespace

```bash
kubectl create namespace k8s-lab
```

---

## Step 2 — Create the YAML

**File:** `05-startup-probe.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: startup-demo
  namespace: k8s-lab

spec:
  replicas: 1

  selector:
    matchLabels:
      app: startup-demo

  template:
    metadata:
      labels:
        app: startup-demo

    spec:
      containers:
      - name: app
        image: busybox:stable

        command:
        - /bin/sh
        - -c
        - |
          echo "Application Starting..."

          sleep 40

          touch /tmp/started

          echo "Application Started"

          while true
          do
            echo "Application Running"
            sleep 5
          done

        startupProbe:
          exec:
            command:
            - cat
            - /tmp/started

          periodSeconds: 5
          failureThreshold: 10

        livenessProbe:
          exec:
            command:
            - cat
            - /tmp/started

          initialDelaySeconds: 5
          periodSeconds: 5
```

**What this YAML does:** `sleep 40` simulates a slow application startup. After 40 seconds, `touch /tmp/started` creates a health file. The Startup Probe repeatedly checks `cat /tmp/started` — until that file exists, the probe fails, but Kubernetes does **not** restart the container while the Startup Probe is still in progress. Once the Startup Probe succeeds, the Liveness Probe becomes active and takes over.

---

## Step 3 — Deploy

```bash
kubectl apply -f 05-startup-probe.yaml
```

**Expected output:**

```text
deployment.apps/startup-demo created
```

---

## Step 4 — Watch the Pod

```bash
kubectl get pods -n k8s-lab -w
```

**Expected output:**

```text
NAME                            READY   STATUS    RESTARTS
startup-demo-xxxxx              1/1     Running   0
```

Notice `RESTARTS = 0`, even though the application takes a full 40 seconds to start.

---

## Step 5 — Describe the Pod

```bash
kubectl describe pod -l app=startup-demo -n k8s-lab
```

Early on, you'll see repeated events like:

```text
Startup probe failed
```

but `RESTARTS` still stays at `0`.

**What's happening — this is the key point of the whole lab:** Normally, a Liveness Probe would have restarted the container by now. Instead, the Startup Probe is effectively telling Kubernetes, "Wait — I'm still starting," and Kubernetes holds off on any Liveness checks until it succeeds.

---

## Step 6 — Wait 40 Seconds

Run the describe command again:

```bash
kubectl describe pod -l app=startup-demo -n k8s-lab
```

You'll now see that the Startup Probe has succeeded, and the application is considered fully started.

---

## Step 7 — Check the Logs

```bash
kubectl logs -l app=startup-demo -n k8s-lab
```

**Expected output:**

```text
Application Starting...
Application Started
Application Running
Application Running
Application Running
```

---

## The Full Timeline

```text
0 sec    → Container Starts
5 sec    → Startup Probe: Fail
10 sec   → Startup Probe: Fail
15 sec   → Startup Probe: Fail
20 sec   → Startup Probe: Fail
25 sec   → Startup Probe: Fail
30 sec   → Startup Probe: Fail
35 sec   → Startup Probe: Fail
40 sec   → Application Finished Starting
         → Startup Probe: Success
         → Liveness Probe Begins
```

The container is never restarted during this entire window — the Startup Probe is protecting it the whole time.

---

## What Happens Without a Startup Probe?

Imagine removing the Startup Probe entirely and keeping only:

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/started

  initialDelaySeconds: 5
```

The timeline would look like this instead:

```text
0 sec   → Container Starts
5 sec   → Liveness Probe: Fail
10 sec  → Liveness Probe: Fail
        → Restart
        → Container Starts Again
5 sec   → Fail
        → Restart
        → Infinite Loop
```

The application would never finish starting — it gets restarted before it ever completes its 40-second boot process.

---

## Startup vs. Liveness — Side by Side

| Startup Probe | Liveness Probe |
|---|---|
| Runs only during application startup | Runs throughout the container's lifetime |
| Protects slow-starting applications | Detects applications that become unhealthy after startup |
| Prevents premature restarts | Restarts unhealthy containers |
| Stops checking once the app has started | Continues checking for the life of the container |

---

## Why This Configuration Works: The Math Behind It

A Startup Probe is *allowed* to fail repeatedly while the application is starting — Kubernetes does not restart the container on every single failure. It only restarts the container if the Startup Probe fails **continuously until the configured failure threshold is reached**.

In this lab's configuration:

```yaml
periodSeconds: 5
failureThreshold: 10
```

Kubernetes tolerates up to `10 failures × 5 seconds = 50 seconds` of startup time before giving up.

Our application finishes in **40 seconds** — comfortably inside that 50-second budget — so it succeeds before the limit is reached, and the Liveness Probe is enabled right after.

## Try It Yourself

Change `sleep 40` to `sleep 60` in the YAML and redeploy.

Work through what should happen before you run it:

- The Startup Probe allows up to 50 seconds (10 × 5) before giving up.
- The application now needs 60 seconds to finish starting.
- So the Startup Probe will exceed its failure threshold **before** the app finishes booting.

Predict the outcome, then apply the change and watch `kubectl get pods -w` to confirm — you should see the container get restarted and the startup cycle begin again from zero. This is a good way to build intuition for how `periodSeconds` and `failureThreshold` work together.

## Cleanup

```bash
kubectl delete -f 05-startup-probe.yaml
```
