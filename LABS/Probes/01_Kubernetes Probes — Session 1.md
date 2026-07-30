# Kubernetes Probes — Session 1: Why Probes Exist

**Build Automate Architect**
*Context → Concept → Lab*

---

## Before We Touch YAML

Most tutorials start probes with YAML.

We are not going to do that.

Because YAML without context is just syntax you memorize and forget.

Instead, we start with a question every engineer eventually asks in production:

> **"My Pod shows `Running`. So why are users still getting errors?"**

If you can answer that question, probes stop being a Kubernetes feature you configure — and become a concept you *understand*.

We'll also connect this session to **Init Containers**, which you've already learned. By the end, you'll see all of these as one continuous story, not four unrelated topics.

---

## Context: The IPL Final on Hotstar

Imagine you're watching the IPL final on Hotstar.

Millions of users are streaming live, at the same time.

Behind the scenes, Hotstar is running thousands of Kubernetes Pods to serve that traffic.

Now imagine **one Pod** develops a problem.

There are exactly three possibilities.

### 1. The Pod is healthy

```
User
   │
Load Balancer
   │
Kubernetes Service
   │
Healthy Pod
```

Traffic flows normally. Nothing to think about here.

### 2. The Pod is running — but broken inside

This is the dangerous one.

Example causes:
- A Java application thread deadlocked
- The database connection pool exhausted
- An infinite loop consuming the event loop
- Memory corruption

The container process itself never crashed. So `kubectl get pods` still says:

```
NAME         STATUS
video-api    Running
```

It *looks* healthy. But real users are getting:

```
500 Internal Server Error
```
```
Connection Timeout
```

**Ask yourself:** Should Kubernetes restart this Pod?

**Answer:** Yes — but here's the catch. Kubernetes doesn't understand Java, Python, or your application logic. All it knows is:

```
Container is Running
```

That's it. That's the entire picture Kubernetes has by default.

So Kubernetes needs a way to ask your application directly:

> *"Are you actually alive, or just technically running?"*

That question is the **Liveness Probe**.

---

## Concept 1: The Liveness Probe

**The question it asks:** *"Are you still alive?"*

| Answer | Kubernetes Action |
|---|---|
| Yes | Nothing happens — leave it alone |
| No | Kill the container, start a new one |

```
Container
    │
Liveness Probe
    │
 Healthy?
  │      │
 Yes     No
  │      │
Keep    Restart
Running Container
```

### Real-world example: Netflix Recommendation Service

Picture Netflix's recommendation engine under heavy load.

One Java thread gets deadlocked. CPU usage looks normal. Memory looks normal. The container is `Running`. But every incoming request just... hangs. Forever.

**Without a liveness probe:**
```
Deadlock occurs → Container keeps "Running" → Serving nobody, forever
```

**With a liveness probe:**
```
Health check fails → Restart → Application comes back healthy
```

This is why Kubernetes is often described as **self-healing**. Liveness probes are the mechanism that makes that true. Without them, a "Running" Pod is just a claim — not a guarantee.

---

## Context: The Slow-Starting Spring Boot App

Now a different scenario.

You deploy a Spring Boot application. Startup time: **90 seconds.**

The container itself starts almost instantly:

```
kubectl get pods
STATUS: Running
```

But under the hood, the application is still:
- Loading in-memory cache
- Loading ML models
- Connecting to Redis
- Connecting to PostgreSQL
- Downloading remote configuration

**Question:** Can users send traffic to this Pod right now?

Obviously not — even though it says `Running`.

### What happens without a Readiness Probe

```
Container Running
      │
Service sends traffic
      │
Application not actually ready
      │
503 Service Unavailable
```

Real users hit real errors, during the exact window your app is still booting.

---

## Concept 2: The Readiness Probe

**The question it asks:** *"Can I send you traffic right now?"*

| Answer | Kubernetes Action |
|---|---|
| Yes | Pod is added as a Service endpoint |
| No | Pod is removed from the Service |

**Critical distinction:** Readiness never restarts or kills the Pod. It only controls whether traffic reaches it.

```
User
   │
Service
   │
Readiness Probe
   │
 Ready?
  │      │
 Yes     No
  │      │
Receive  Don't Receive
Traffic  Traffic
```

### Real-world example: Hotstar during IPL

A new Pod spins up during the match to handle a traffic surge. It needs **45 seconds** to load video metadata before it can serve anything correctly.

If traffic hits it immediately:

```
500 / 503 / Timeout
```

Instead, the Readiness Probe reports `NOT READY` for those 45 seconds. The Service simply ignores that Pod — routes traffic elsewhere — until it reports `READY`. Users never notice a thing.

---

## Liveness vs. Readiness — The Table Worth Memorizing

| | **Liveness** | **Readiness** |
|---|---|---|
| Question | Is the application alive? | Can the application receive traffic? |
| On failure | Restart the Pod | Remove Pod from Service (no restart) |
| Purpose | Self-healing | Traffic control |
| Effect on container | Restarts it | Never restarts it |

**Memory anchor:** Liveness protects the *application*. Readiness protects the *users*.

---

## Connecting Back to Init Containers

This is where the full lifecycle finally clicks into place.

```
Init Container
      │
Main Container Starts
      │
Readiness Probe
      │
Traffic Starts
      │
Liveness Probe
      │
Continuous Monitoring
```

In plain English, each stage is making a different statement:

- **Init Container:** "I am preparing the environment."
- **Readiness Probe:** "I am ready to serve."
- **Liveness Probe:** "I am still healthy — check on me continuously."

---

## Context: When Even Readiness and Liveness Aren't Enough

Modern applications have gotten heavier.

Think about:
- Spring Boot apps loading large ML models
- TensorFlow serving containers
- Elasticsearch nodes
- LLM inference backends

Some of these can take **3+ minutes** just to boot.

**The problem:** Liveness probing starts checking immediately by default. While the app is still legitimately starting up, it reports "not ready yet" — and Liveness misinterprets that as "it's dead."

```
Start → Liveness checks → Fails → Restart → Starts again → Fails again → Restart again...
```

The application enters a **restart loop** and never actually finishes booting. This was a real, recurring production problem before Kubernetes introduced a third probe.

---

## Concept 3: The Startup Probe

**The question it asks:** *"Have you finished starting up yet?"*

Startup Probe runs *only* during the boot phase. While it's active, Liveness checks are held off entirely.

```
Startup Probe
      │
  Succeeds
      │
Liveness Enabled
      │
Readiness Enabled
```

### Real-world example: An LLM Backend (like a ChatGPT-style service)

Imagine a backend that needs to load:
- A 15 GB model
- Embeddings
- Tokenizer
- Local cache

Boot time: roughly 2 minutes.

**Without a Startup Probe:**
```
Start → Liveness kicks in early → Fails → Restart → Fails again → Restart again
```
The application never gets the chance to finish loading — it's killed mid-boot, every time.

**With a Startup Probe:** Kubernetes waits patiently until startup genuinely succeeds, *then* switches on Liveness and Readiness.

---

## All Three Probes, Side by Side

Think of them as three different questions Kubernetes is asking your application, at three different moments:

| Probe | Question | When it runs | Failure action |
|---|---|---|---|
| **Startup** | "Have you finished starting?" | Only during boot | Kills and restarts if it never succeeds |
| **Readiness** | "Can users send requests now?" | Continuously | Removes Pod from Service |
| **Liveness** | "Are you still healthy?" | Continuously (after Startup succeeds) | Restarts the container |

---

## The Complete Pod Lifecycle

```
Pod Created
      │
Init Container
      │
Main Container Starts
      │
Startup Probe
      │
Application Started
      │
Readiness Probe
      │
Service Starts Routing Traffic
      │
Liveness Probe
      │
Continuous Monitoring
```

---

## One-Line Memory Trick

Say each of these to yourself as a sentence spoken *by the Pod*:

- **Init Container** → "Prepare me."
- **Startup Probe** → "Wait, I'm still starting."
- **Readiness Probe** → "I'm ready for users."
- **Liveness Probe** → "I'm still alive."

Four sentences. One lifecycle. Not four unrelated Kubernetes features.

---

## Recap: What We Covered Today

1. Why `Running` does **not** mean **Healthy**
2. **Liveness Probe** — self-healing for broken-but-running containers
3. Why `Running` does **not** mean **Ready**
4. **Readiness Probe** — traffic control during startup and temporary unavailability
5. How both connect with **Init Containers** to form one lifecycle
6. **Startup Probe** — protecting slow-booting applications from premature restarts
7. The full Pod lifecycle, end to end

---

**Next Session:** We move from *why* probes exist to *how* to configure them — writing the actual YAML for `livenessProbe`, `readinessProbe`, and `startupProbe`, followed by a hands-on EKS lab.
