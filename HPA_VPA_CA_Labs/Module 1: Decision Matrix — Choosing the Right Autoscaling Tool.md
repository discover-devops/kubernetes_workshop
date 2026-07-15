# Module 1: Decision Matrix — Choosing the Right Autoscaling Tool

**Format:** Concept-first, with verification exercises against your existing cluster (no new infrastructure — you already built and ran HPA, VPA, and Cluster Autoscaler in the earlier lab series; this module is about **deciding correctly**, not building again)


**Prerequisites:** Completed the HPA, VPA, and Cluster Autoscaler lab series on `scaling-demo-cluster`

---

## The Running Analogy: Hotstar Streaming the IPL Final

Think about what actually happens inside Hotstar/Disney+ Hotstar's infrastructure on IPL Final night, when **59+ million concurrent viewers** hit the platform at the exact same moment the toss happens. This is a real, famous, publicly-discussed scaling event in the Indian tech industry — and every autoscaling concept in this module maps onto a real decision Hotstar's engineering team actually had to make.

- **HPA** is the layer that decides: *"how many API server replicas do I need right now, this minute, to handle the concurrent viewer count?"*
- **VPA** is the layer that decides: *"is each individual video-transcoding pod correctly sized, or is it starved for CPU / wasting memory?"*
- **Cluster Autoscaler / Karpenter** is the layer that decides: *"do I even have enough physical machines in my cluster to run all these pods, and if not, how fast can I get more?"*

Three completely different questions. Three completely different tools. Picking the wrong tool for the wrong question is where production incidents come from — and that's exactly what this module trains you to avoid.

---

## Part 1: When to Use HPA — "Scale the Number of Streams Handling Requests"

### The scenario

The toss happens. Viewer count goes from 2 million to 40 million in **90 seconds**. This is a *predictable, cyclical* spike — Hotstar's engineers know match days cause this, they know roughly when it happens (toss time, key overs, wickets), and every single request-handling pod is identical and stateless — it doesn't matter *which* replica of the video-manifest API serves your request.

This is a textbook HPA scenario: **stateless, horizontally identical, traffic-driven.**

### Use HPA when:

- The application is **stateless** — any replica can serve any request, no "sticky" data
- Traffic is **predictable and cyclical** (daily peak hours, live sporting events, sale-day e-commerce traffic)
- Pods are **already correctly resource-sized** (HPA scales *count*, not *size* — feeding it undersized pods just multiplies the problem)
- You need **fine-grained control** over scaling triggers (CPU, memory, or custom metrics like requests-per-second)
- The app can **handle pod restarts gracefully** — no in-memory session state that would be lost

### Example workloads

REST APIs, web frontends, the video-manifest/playback-URL service, stateless microservices, queue workers.

### Don't use HPA when

- The application is **stateful** (a live-transcoding session tied to a specific pod — use a StatefulSet instead, with far more care)
- A **single replica is genuinely sufficient** — HPA adds operational complexity for zero benefit here
- **Startup time is very long** (minutes) — by the time a new replica is ready, the traffic spike that triggered it may have already passed. HPA is reactive; it can't scale for a spike it hasn't seen yet.

### Scaling behavior — stabilization windows (the anti-flapping mechanism)

Recall from your HPA lab: the controller doesn't scale instantly on every fluctuation. It uses **stabilization windows**:

| Direction | Default window | Why |
|---|---|---|
| Scale **up** | 0 seconds (immediate) | When 40 million viewers show up, you want replicas *now*, not after a cooldown |
| Scale **down** | 300 seconds (5 minutes) | Prevents "the 4th over just ended, traffic dipped for 20 seconds, HPA scaled down, then a wicket fell and traffic spiked back up" — flapping |

This asymmetry is deliberate: **scaling up fast is cheap and safe; scaling down fast is risky and can cause thrashing.**

###  Verification Exercise — check this on your own cluster

You already have HPA experience from the earlier lab series. Let's verify you can read this behavior directly from a live object, not just recall it:

```bash
kubectl get hpa php-apache-hpa -o yaml 2>/dev/null | grep -A 10 "behavior:" 
```

If you've cleaned up that HPA already, recreate a minimal one just to inspect the field (no load generation needed — we're just reading the object's defaults):

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=5 2>/dev/null
kubectl get hpa php-apache -o jsonpath='{.spec.behavior}' | jq . 2>/dev/null || echo "Default behavior not explicitly set — controller uses built-in defaults (0s up / 300s down)"
kubectl delete hpa php-apache --ignore-not-found
```

>  If no explicit `behavior` block is set (common), the controller still applies the 0s/300s defaults internally — they just won't show in the YAML. This is a subtlety worth knowing: **absence of the field doesn't mean absence of the behavior.**

---

## Part 2: When to Use VPA — "Right-Size Each Individual Video-Processing Pod"

### The scenario

Separately from viewer-facing traffic, Hotstar runs a fleet of **video transcoding pods** — converting the raw camera feed into multiple resolutions (240p through 4K) for adaptive bitrate streaming. These pods aren't about handling *more concurrent users* — there's a fixed number of camera angles and resolution outputs. The question isn't "how many transcoders do I need," it's **"is each transcoder pod actually sized correctly for the CPU/memory a 4K transcode job really needs?"**

This is a textbook VPA scenario: **resource sizing is uncertain, workload characteristics vary, and horizontal replica count isn't the lever that matters.**

###  Critical Warning — internalize this before anything else in this section

**VPA evicts pods to resize them.** In `Auto` mode (recall Lab 2), this means the pod is killed and recreated with new resource values — not a live, in-place resize (true in-place resize is a separate alpha Kubernetes feature requiring 1.33+; your cluster is on 1.30). If you're running a **single replica** of something VPA is actively resizing, that eviction **is downtime**, full stop.

### The VPA + HPA conflict — the single most common production misconfiguration

If HPA and VPA both target the **same metric** (say, both watching CPU) on the same workload, they enter a fight:

```
VPA sees high CPU → evicts pod to give it more CPU request
        ↓
HPA sees a pod disappear + reappear with different resource shape → reacts by changing replica count
        ↓
The changed replica count changes per-pod CPU load
        ↓
VPA sees "new" CPU behavior → evicts again
        ↓
(repeat, indefinitely)
```

### The safe combination matrix

| Scenario | Safe? | Why |
|---|---|---|
| HPA on CPU + VPA on Memory |  Safe | Different metrics, no feedback loop |
| HPA on custom metrics (e.g., requests/sec) + VPA on CPU/Memory |  Safe | No overlap at all |
| HPA on CPU + VPA on CPU |  Conflict | Both reacting to the same signal, feedback loop |
| HPA on Memory + VPA on Memory |  Conflict | Same problem, different metric |

**Rule of thumb:** if you must run both on the same workload, keep them on strictly separate metrics — or run VPA in `Off` mode (recommendation-only, exactly like Step 5 of your Lab 2) so it never actually evicts anything, and let HPA own all real scaling action.

### Use VPA when

- Resource requirements **vary significantly** over time or across pod types (a 240p transcode vs. a 4K transcode has wildly different CPU needs)
- You want to **reduce resource waste** from over-provisioned guesses
- Memory usage **grows with workload** in a way that's hard to predict upfront
- You're **genuinely unsure** of correct sizing and want data-driven recommendations first (`Off` mode)

### Example workloads

Batch processing jobs, ML inference services, the video transcoding fleet, data analytics jobs.

### Don't use VPA when

- The application **cannot tolerate restarts** at all (use `Off` mode for recommendations only — never `Auto`)
- Resources are **already optimally sized** (VPA adds churn for no benefit)
- The workload is **extremely predictable** (a fixed, well-known resource footprint doesn't need a statistical recommender)
- You're **already running HPA on the same metric** (conflict, as above)

### 🔧 Verification Exercise

If VPA is still installed from your Lab 2 (or reinstall per that lab's Step 2), inspect the actual `updateMode` field live, and connect it to the eviction risk described above:

```bash
kubectl get vpa -A -o custom-columns='NAME:.metadata.name,MODE:.spec.updatePolicy.updateMode' 2>/dev/null || echo "No VPA objects currently present — that's fine, this is a read-only check"
```

If you have one in `Auto` mode targeting a single-replica Deployment, that's a live example of exactly the downtime risk called out above — worth flagging as an anti-pattern if you see it.

---

## Part 3: When to Use Cluster Autoscaler — "Get More Standard Servers When the Fleet Runs Out"

### The scenario

HPA has scaled the video-manifest API to 40 replicas. VPA has right-sized the transcoders. But now there's a harder problem: **the underlying EC2 fleet itself doesn't have enough CPU/memory capacity to actually run all these pods.** Someone needs to talk to AWS and provision more physical machines. That's Cluster Autoscaler's job — and recall from your Lab 3, it can only launch **more of the same pre-configured instance type** your node group's ASG was built with.

### How Cluster Autoscaler actually works (from your own Lab 3 logs)

```
Pod created → Pending (unschedulable)
        ↓
CA checks for pending pods every 10 seconds
        ↓
CA calculates how many nodes (of its ONE available type) are needed
        ↓
Cloud provider API called (AWS Auto Scaling Group SetDesiredCapacity)
        ↓
Node provisioned — 2 to 5 minutes (cloud API call + EC2 boot + kubelet registration)
        ↓
Pod scheduled
```

### Scale-down logic

- CA checks node utilization every 10 seconds
- A node below the utilization threshold for a sustained period (default 10 minutes) is marked for removal
- CA **simulates** rescheduling all of that node's pods elsewhere first, to confirm it's safe
- Only then does it drain and terminate the node

### Use CA when

- Running on **managed Kubernetes** (EKS, GKE, AKS) — CA is cloud-agnostic, unlike Karpenter
- You need **simple, proven** autoscaling with a long production track record
- Node types are **standardized** — your workloads don't need wildly different instance shapes
- **Budget predictability** matters more than absolute cost optimization

### Example scenarios

Dev/staging environments, small-to-medium production clusters, cost-sensitive environments running mostly on-demand.

### Don't use CA when

- You need **sub-minute** scaling (CA's 2-5 minute latency is a hard architectural floor, not a tuning knob)
- Workloads require **genuinely diverse instance types** (CA can only pick from what's in its pre-defined node groups — this is the exact limitation your Lab 3 demonstrated when it launched 2× identical `t3.medium` nodes because that was the only option available)
- **Aggressive cost optimization** is the priority (no dynamic instance-type selection, no automatic Spot diversification the way Karpenter does)

### 🔧 Verification Exercise — reconnect this to your actual Lab 3 evidence

```bash
eksctl get nodegroup --cluster scaling-demo-cluster --region ap-south-1
```

Look at the `INSTANCE TYPE` column — this is CA's entire "menu," exactly as discussed in the original lab. There is no live CA installed right now (you uninstalled it after Lab 3) — this command just confirms the *structural* limitation still described above: whatever instance type is in this node group is the **only** type CA could ever have launched, regardless of what any given pod actually needed.

---

## Part 4: Karpenter as the Contrast Case — "The Dynamic Fleet"

*(Full depth already covered in your Karpenter lab; this section is intentionally brief — a bridge, not a repeat.)*

Where CA is locked to a pre-built menu of instance types, Karpenter — as your Lab 4 demonstrated directly — evaluates the **actual, real-time shape** of pending pods and picks the best-fitting instance type from the full EC2 catalog, live, per batch of pending pods. Recall your own numbers: CA launched 2× identical `t3.medium` for 6 pending pods (no choice available); Karpenter launched 2× `t3.xlarge` for 8 pending pods needing more capacity per node — a decision CA structurally could never make.

### Quick comparison table

| Feature | Cluster Autoscaler | Karpenter |
|---|---|---|
| Scaling speed | 2-5 minutes (cloud API + boot) | Under 1 minute typically |
| Node flexibility | Rigid — pre-defined node groups only | Dynamic — picks from full EC2 catalog per decision |
| Cloud support | Multi-cloud (AWS, GCP, Azure) | AWS-only |
| Cost strategy | Predictable, standard on-demand | Optimized — Spot-aware, exact-fit sizing |
| Ideal cluster size | Small-medium, dev/staging | Large-scale production, ML, batch |
| Team overhead | Low, simple, proven | Requires AWS-specific expertise |

### Why Karpenter is structurally smarter, not just faster

Cluster Autoscaler is **reactive** — it only acts once pods are already `Pending`. Karpenter performs what's often called **advanced bin-packing**: it looks at the *entire batch* of currently-pending pods simultaneously, calculates their combined CPU/memory footprint, and searches the EC2 catalog for the instance type combination that minimizes wasted capacity ("bin waste") at the lowest cost — not just "launch one more of whatever I already have."

---

## The Decision Matrix — Put It All Together

| Question you're actually asking | Tool | Hotstar example |
|---|---|---|
| "Do I have enough **replicas** of this stateless service right now?" | **HPA** | Video-manifest API during toss-time traffic spike |
| "Is each **individual pod** sized correctly for CPU/memory?" | **VPA** | 4K transcoder pod resource footprint |
| "Do I have enough **standard machines** in my fleet?" | **Cluster Autoscaler** | Steady-state capacity growth on a predictable, standardized fleet |
| "Do I need the **right machine, right now, fast**, for a highly variable/bursty workload?" | **Karpenter** | IPL Final night, unpredictable multi-fold traffic spike, cost-optimized Spot usage |

### The meta-rule

These four tools **answer different questions and operate on different axes** (pod count vs. pod size vs. node count vs. node type). They are not mutually exclusive alternatives — a real production Hotstar-scale system runs **HPA + VPA (careful, separate metrics) + Karpenter (or CA) simultaneously**, each solving its own layer of the problem. The skill being tested here isn't "which one tool is best" — it's **correctly diagnosing which layer a given scaling problem actually lives in.**

---

## Interview Questions

**Q1. Your HPA is scaling replicas correctly, but overall response latency is still degrading under load. What could be wrong, and which tool(s) would you investigate?**
> *Expects: candidate should reason that HPA only controls replica count — if individual pods are CPU-throttled due to undersized resource requests, more replicas won't fix per-pod slowness. Points toward VPA (or manual resource tuning) as the missing layer, and possibly node-level capacity (CA/Karpenter) if pods are even getting scheduled at all.*

**Q2. Explain, in your own words, exactly why HPA and VPA conflict when targeting the same metric. Don't just say "they fight" — describe the actual mechanical loop.**
> *Expects: the eviction → replica-count-change → new-per-pod-load → new-eviction loop, as detailed above. Tests whether the candidate understands mechanism, not just the rule.*

**Q3. You're running a single-replica Postgres pod with VPA in `Auto` mode. What's wrong with this setup, and how would you fix it?**
> *Expects: VPA evictions cause real downtime on single-replica stateful workloads. Fix: either run VPA in `Off` mode for recommendations only, or ensure adequate replication/failover exists before allowing `Auto` mode evictions — and note that VPA + StatefulSets/databases requires extra care generally.*

**Q4. Cluster Autoscaler takes 2-5 minutes to add capacity. Name two concrete engineering mitigations for this latency, other than switching to Karpenter.**
> *Expects: over-provisioning (keeping some buffer/slack pods or nodes ready ahead of demand — e.g., low-priority "placeholder" pods that get preempted), and predictive/scheduled scaling ahead of known traffic events (e.g., scaling up ahead of a known IPL match start time rather than reactively).*

**Q5. A workload needs GPU instances for some jobs and standard CPU instances for others, within the same cluster, changing week to week. Which autoscaler fits better, and why?**
> *Expects: Karpenter — CA would require manually pre-defining and maintaining separate node groups per instance shape, which doesn't adapt automatically; Karpenter's NodePool can express multiple instance type constraints and pick dynamically per pod's actual requirements.*

**Q6. What does "stabilization window" mean for HPA, and why is the default asymmetric (0s up / 300s down)?**
> *Expects: prevents flapping — scaling up fast is safe/cheap, scaling down fast risks prematurely removing capacity right as a new spike begins. Should connect to a concrete flapping scenario (e.g., a brief lull between overs in a cricket match).*

---

## Troubleshooting Scenarios

**Scenario A:** Your HPA shows `TARGETS: <unknown>/50%` and refuses to scale, even under clear load.
> *Diagnosis path: Metrics Server not installed/healthy, or the target Deployment's containers are missing `resources.requests` (HPA cannot compute a percentage without a request baseline — recall Lab 1, Step 5). Check `kubectl top pods` first; if that also fails, it's a Metrics Server problem, not an HPA problem.*

**Scenario B:** VPA's `Target` recommendation is wildly higher than what `kubectl top pod` shows for live usage.
> *Diagnosis path: not necessarily a bug — VPA's confidence-multiplier algorithm inflates recommendations early in a NodePool's/VPA object's life (few samples = wide safety margin), and memory recommendations specifically track peak usage, not average, because OOM is unrecoverable. Recheck after the VPA object has run longer and accumulated more samples before assuming misconfiguration.*

**Scenario C:** Cluster Autoscaler logs show `Skipping <node> - node group min size reached`, and pods stay `Pending`.
> *Diagnosis path: this is CA correctly refusing to scale below the ASG's configured `minSize` — check if the real problem is actually a *scale-up* failure elsewhere (e.g., `maxSize` reached, or missing ASG auto-discovery tags), not a scale-down issue. Read the log message literally before assuming something is broken.*

**Scenario D:** You migrated from CA to Karpenter, but pods are still `Pending` and no new nodes appear.
> *Diagnosis path: check for the classic Karpenter setup gaps from Lab 4 — subnet/security-group discovery tags missing, `EC2NodeClass` not `Ready`, or the NodePool's `instance-type` requirements too narrow to fit the pending pod's resource request.*

---

## Best Practices Checklist

-  Never target the same metric with HPA and VPA on the same workload
-  Always set `resources.requests` before enabling HPA — it's the mathematical baseline HPA depends on
-  Use VPA in `Off` mode first to gather recommendations before ever enabling `Auto` mode in production
-  For single-replica stateful workloads, avoid VPA `Auto` mode entirely, or ensure failover exists first
-  Match your autoscaler to your cloud reality: multi-cloud → CA; AWS-only + need speed/cost optimization → Karpenter
-  Treat CA's 2-5 minute latency as an architectural constant, not a tuning problem — plan around it (over-provisioning, predictive scaling) rather than fighting it

## Common Pitfalls

-  Assuming "more replicas" (HPA) fixes a problem that's actually "pods are undersized" (VPA territory)
-  Running VPA `Auto` mode on anything without redundancy
-  Expecting Cluster Autoscaler to pick a "better" instance type — it structurally cannot; that's Karpenter's job
-  Forgetting that HPA's scale-down stabilization window is 5 minutes by default — mistaking this delay for a malfunction
-  Deploying Karpenter without properly tagging subnets/security groups for discovery, then assuming the controller itself is broken when it's actually a configuration gap

---

*End of Module 1. Proceed to Module 2: Network Policies — The Security Fortress, once this decision framework is comfortable.*
