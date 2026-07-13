# Lab 2: Vertical Pod Autoscaler (VPA) on Amazon EKS

**Level:** Intermediate
**Duration:** ~45–60 minutes
**Prerequisites:** Completed **Lab 1 (HPA)**, with `scaling-demo-cluster` still running and cleaned up (no leftover HPA/php-apache resources)

---

## What You'll Learn

- The difference between horizontal scaling (HPA) and vertical scaling (VPA)
- Why VPA and HPA should never control the same resource on the same workload
- How to install VPA (it's not a core Kubernetes API — separate project, separate install)
- The three VPA components — Recommender, Updater, Admission Controller — and how they interact
- How to read a VPA recommendation (Lower Bound / Target / Upper Bound)
- Why VPA's CPU and memory algorithms are deliberately different
- How to safely roll out VPA using `Off` mode before `Auto` mode
- What actually happens on disk/in the cluster when VPA "applies" a recommendation

---

## Core Concept: HPA vs. VPA

| | HPA | VPA |
|---|---|---|
| **Scales** | Number of pods (horizontal) | Resources per pod — CPU/memory (vertical) |
| **Core K8s API?** | Yes (`autoscaling/v2`) | No — separate project (`kubernetes/autoscaler`) |
| **Install method** | Built-in | Clone repo + run install script |
| **Good for** | Stateless apps under variable traffic | Right-sizing apps where correct CPU/memory is hard to guess upfront |

>  **Important:** Never run HPA and VPA on the same resource (e.g., both controlling CPU) for the same Deployment — they will fight each other. This is why Lab 2 uses a **separate Deployment** (`vpa-demo-app`), not the `php-apache` Deployment from Lab 1.

---

## Step 1 — Check Prerequisites

VPA's install script (`vpa-up.sh`) requires `git` and a modern `openssl` (≥1.1.1, needs `-addext` support).

```bash
git --version
openssl version
```

If `git` is missing:

```bash
sudo dnf install -y git
```

Confirm Metrics Server (installed in Lab 1) is still healthy — VPA's Recommender depends on it, same as HPA does:

```bash
kubectl top nodes
```

Also confirm no leftover resources from Lab 1:

```bash
kubectl get pod
```

Should return `No resources found in default namespace` (or only resources you intend to keep). Clean up anything stray before continuing.

---

## Step 2 — Install VPA

VPA is not distributed as a single manifest or an EKS add-on. It's installed by cloning the `kubernetes/autoscaler` repo and running its install script.

```bash
cd ~
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

⏱️ Takes 2–3 minutes. This script:
- Applies the `VerticalPodAutoscaler` CRD (`autoscaling.k8s.io/v1`)
- Deploys three components into `kube-system`: **Recommender**, **Updater**, **Admission Controller**
- Generates and uploads a self-signed CA cert/secret used by the Admission Controller webhook

### Verify all three components are running:

```bash
kubectl get pods -n kube-system | grep vpa
```

**Expected output** (suffixes will differ):

```
vpa-admission-controller-xxxxxxxxx-xxxxx   1/1   Running   0   90s
vpa-recommender-xxxxxxxxx-xxxxx            1/1   Running   0   90s
vpa-updater-xxxxxxxxx-xxxxx                1/1   Running   0   90s
```

All three must show `1/1 Running` before proceeding.

---

## Understanding the Three VPA Components

This is the core mental model for the rest of the lab.

### 1. Recommender
- Watches resource usage (CPU/memory) for pods matched by a `VerticalPodAutoscaler` object's `targetRef`, pulling data from the Metrics API (same source HPA uses).
- Maintains a **rolling statistical model** of usage over time — not just a live snapshot.
- Outputs three numbers per container: **Lower Bound**, **Target**, **Upper Bound**.
- Only **calculates** and writes recommendations into the VPA object's `.status` field. It never touches a running pod directly.

### 2. Updater
- Periodically compares running pods' actual resource requests against the Recommender's current recommendation.
- If they differ significantly (and `updateMode` allows it), the Updater **evicts** the pod.
- Does not create the replacement — the owning Deployment/ReplicaSet notices the missing pod and creates a new one through normal reconciliation.
- Only active when `updateMode` is `"Auto"` or `"Recreate"`. In `"Off"` mode, it does nothing.
- Processes pods **one at a time**, not all replicas simultaneously, to avoid an availability gap.

### 3. Admission Controller
- A **mutating webhook** that intercepts pod creation requests — any pod creation, whether from a fresh `kubectl apply` or from an Updater-triggered eviction/replacement.
- Checks whether the pod belongs to a VPA target, and if so, rewrites `resources.requests` (and `limits`, maintaining the original ratio) to match the latest recommendation, **before the pod starts running**.
- This is the only component that actually writes new values onto a pod spec.

### The full sequence:

```
Recommender  →  writes recommendation into VPA object status
                        ↓
Updater  →  sees live pod resources ≠ recommendation → evicts pod (one at a time)
                        ↓
Deployment controller  →  notices missing pod → creates replacement
                        ↓
Admission Controller  →  intercepts new pod creation → injects recommended resources
                        ↓
New pod starts running with right-sized resources
```

It's **event-driven**, not a fixed schedule — the Recommender always runs in the background, but the Updater/Admission Controller only act when the mode permits it and an eviction/creation event actually happens.

---

## Step 3 — Deploy a Deliberately Under-Resourced Demo App

We'll intentionally set the CPU/memory requests too low, so VPA has something clear to correct.

```bash
cat <<'EOF' > vpa-demo-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vpa-demo-app
  labels:
    app: vpa-demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vpa-demo-app
  template:
    metadata:
      labels:
        app: vpa-demo-app
    spec:
      containers:
      - name: vpa-demo-app
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 50m
            memory: 50Mi
          limits:
            cpu: 100m
            memory: 100Mi
---
apiVersion: v1
kind: Service
metadata:
  name: vpa-demo-app
  labels:
    app: vpa-demo-app
spec:
  ports:
  - port: 80
  selector:
    app: vpa-demo-app
EOF

kubectl apply -f vpa-demo-app.yaml
```

### Verify:

```bash
kubectl get deployment vpa-demo-app
kubectl get pods -l app=vpa-demo-app -o wide
```

Both pods should show `Running`, `1/1`.

---

## Step 4 — Create the VPA Object in `Off` Mode (Recommendation-Only)

`Off` mode means only the Recommender is active — we can see what VPA calculates without any pod disruption yet.

```bash
cat <<'EOF' > vpa-demo-app-vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-demo-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vpa-demo-app
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
    - containerName: vpa-demo-app
      minAllowed:
        cpu: 25m
        memory: 32Mi
      maxAllowed:
        cpu: 1000m
        memory: 512Mi
      controlledResources: ["cpu", "memory"]
EOF

kubectl apply -f vpa-demo-app-vpa.yaml
kubectl get vpa vpa-demo-app-vpa
```

> 📝 `minAllowed` / `maxAllowed` bound the Recommender's output, similar in spirit to `minReplicas`/`maxReplicas` on HPA — prevents recommending something absurdly small or large.

---

## Step 5 — Generate Load

```bash
for i in 1 2 3; do
  kubectl run vpa-load-$i --image=busybox:1.36 --restart=Never -- /bin/sh -c \
    "while true; do wget -q -O- http://vpa-demo-app.default.svc.cluster.local; done"
done
```

---

## Step 6 — Read the Recommendation

Unlike HPA's 15-second cycle, the Recommender needs several minutes of observed usage to build a confident statistical model. Wait **3–5 minutes**, then:

```bash
kubectl describe vpa vpa-demo-app-vpa
```

Look at the `Status → Recommendation → Container Recommendations` block. Example output:

```
Recommendation:
  Container Recommendations:
    Container Name:  vpa-demo-app
    Lower Bound:
      Cpu:     25m
      Memory:  250Mi
    Target:
      Cpu:     93m
      Memory:  250Mi
    Uncapped Target:
      Cpu:     93m
      Memory:  250Mi
    Upper Bound:
      Cpu:     1
      Memory:  512Mi
```

### How to read this against your original manifest:

| | Original (our guess, in Deployment YAML) | VPA Target (calculated, in VPA object status) |
|---|---|---|
| CPU | `50m` | e.g. `93m` |
| Memory | `50Mi` | e.g. `250Mi` |

VPA is saying: *"You requested 50m/50Mi. Based on what I observed this app actually consume under load, it needs roughly this much to run comfortably."*

### Why CPU and memory recommendations behave so differently

Both come from the same Recommender, but use different math because the two resources **fail differently**:

- **CPU** uses a decaying histogram of usage samples, targeting roughly the 90th percentile. CPU spikes are tolerable — the kernel just throttles the container briefly; the app slows but doesn't crash. So VPA can size close to typical usage.
- **Memory** uses something closer to **peak observed usage** plus a safety margin. There's no graceful degradation for memory — exceeding a limit means the kernel's OOM killer terminates the container immediately. VPA is deliberately conservative here because the cost of under-recommending is asymmetric (a wasted request vs. a crashed pod).

### Why the recommendation may look much higher than live `kubectl top pod` usage

```bash
kubectl top pod
```

You may see live usage (e.g., `57m`/`14Mi`) well below the VPA `Target` (e.g., `93m`/`250Mi`), especially early on. This is because the Recommender applies a **confidence multiplier** that shrinks as more samples accumulate:

```
recommendation ≈ statisticalEstimate × (1 + confidenceMultiplier / sqrt(numberOfSamples))
```

With a young VPA object and few samples, the multiplier inflates the recommendation as a safety buffer. As more time passes and samples accumulate, this converges closer to true observed usage. Memory additionally has a longer decay half-life than CPU (again, because forgetting a past spike too quickly risks a future OOM), so memory recommendations can stay elevated longer than CPU ones.

> 💡 **Optional:** re-run `kubectl describe vpa` every 10–15 minutes to watch the `Target` values evolve as more samples accumulate.

---

## Step 7 — Switch to `Auto` Mode and Watch It Apply Live

```bash
kubectl patch vpa vpa-demo-app-vpa --type='merge' -p '{"spec":{"updatePolicy":{"updateMode":"Auto"}}}'
```

### Terminal 1 — watch pods get evicted and recreated:

```bash
kubectl get pods -l app=vpa-demo-app --watch
```

Expected pattern:

```
vpa-demo-app-xxxxx-aaaaa   1/1   Terminating         0   32m
vpa-demo-app-xxxxx-bbbbb   0/1   Pending             0   0s
vpa-demo-app-xxxxx-bbbbb   0/1   ContainerCreating   0   1s
vpa-demo-app-xxxxx-bbbbb   1/1   Running             0   4s
```

> 📝 Note the **pod name changes** on replacement — this confirms genuine evict-and-recreate behavior (this cluster is on Kubernetes 1.30; true in-place resize without pod restart is a separate alpha feature requiring 1.33+). The Updater also processes replicas **one at a time**, not all at once — so during the transition you'll briefly see one pod already updated while its sibling still shows old values.

### Terminal 2 — inspect injected resources once pods settle:

```bash
kubectl get pods -l app=vpa-demo-app -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].resources}{"\n"}{end}'
```

**Expected output** — both pods should now carry matching, VPA-corrected values, e.g.:

```
vpa-demo-app-xxxxx-aaaaa   {"limits":{"cpu":"252m","memory":"500Mi"},"requests":{"cpu":"126m","memory":"250Mi"}}
vpa-demo-app-xxxxx-bbbbb   {"limits":{"cpu":"252m","memory":"500Mi"},"requests":{"cpu":"126m","memory":"250Mi"}}
```

>  **Notice the limit-to-request ratio is preserved.** Original manifest had `limits:requests = 2:1` (`100m:50m`, `100Mi:50Mi`). The new values maintain that same 2:1 ratio (e.g., `252m:126m`, `500Mi:250Mi`) rather than just overwriting the request and leaving the limit untouched. This is VPA's default `controlledValues` behavior — it scales both together to preserve your originally defined ratio.

This confirms the full pipeline worked end-to-end: **Recommender calculated → Updater evicted (one pod at a time) → Admission Controller injected new values into each replacement → cluster converged to a consistent, right-sized state.**

---

## Step 8 — Cleanup (Prepare for Lab 3)

Run in order — this tears down Lab 2 resources including the cluster-wide VPA components, so Lab 3 starts clean. **The EKS cluster itself is left running.**

### 8a. Stop load generators:

```bash
for i in 1 2 3; do kubectl delete pod vpa-load-$i --ignore-not-found; done
```

### 8b. Remove the VPA object:

```bash
kubectl delete vpa vpa-demo-app-vpa
```

### 8c. Remove the demo app:

```bash
kubectl delete -f vpa-demo-app.yaml
```

### 8d. Uninstall the VPA components:

VPA's Recommender/Updater/Admission Controller are cluster-wide (installed in `kube-system`), not scoped to one app — remove them between labs to keep the cluster state clean.

```bash
cd ~/autoscaler/vertical-pod-autoscaler
./hack/vpa-down.sh
```

### 8e. Verify clean state:

```bash
kubectl get vpa
kubectl get pods -l app=vpa-demo-app
kubectl get pods -n kube-system | grep vpa
```

All three should return empty / `No resources found`.

>  **Keep the cluster running** (`scaling-demo-cluster`) — Lab 3 (Cluster Autoscaler) reuses it. Do **not** run `eksctl delete cluster` yet.

---

## Troubleshooting Quick Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| `vpa-up.sh` fails with an openssl `-addext` error | OpenSSL too old | Upgrade to 1.1.1+ (check with `openssl version`) |
| `git: command not found` | Not installed on base AMI | `sudo dnf install -y git` |
| `RecommendationProvided` missing / no Target values | Not enough time/samples yet | Wait 3–5 min after applying load; Recommender needs time to build its model |
| Memory `Target` seems much higher than `kubectl top pod` shows | Expected — confidence multiplier + peak-based algorithm | Wait longer for more samples; will converge down over time |
| Pods not evicting in `Auto` mode | `updateMode` still `Off`, or Updater not seeing significant enough difference | Confirm patch applied: `kubectl get vpa -o yaml \| grep updateMode` |
| Only one pod updated, sibling still has old resources | Expected — Updater processes pods one at a time | Wait a few more seconds/minutes for the next eviction cycle |
| VPA and HPA both configured on same Deployment/resource | Anti-pattern — they will conflict | Never control the same resource (e.g. CPU) with both on one workload |

---

*End of Lab 2. Proceed to Lab 3: Cluster Autoscaler (CA) once cleanup is verified.*
