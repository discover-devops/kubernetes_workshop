# Kubernetes Probes — Summary

**Build Automate Architect**
*Context → Concept → Lab*

A quick-reference recap of everything covered in this session on Startup, Readiness, and Liveness Probes.

---

## The Three Probes at a Glance

| Probe | Purpose | Runs When? | Failure Action | Typical Use Case |
|---|---|---|---|---|
| **Startup Probe** | Check if the application has finished starting | Only during container startup | Restart container **only if it fails beyond `failureThreshold`** | Slow-starting apps like Spring Boot, Elasticsearch, ML models |
| **Readiness Probe** | Check if the application is ready to receive traffic | Continuously, after startup | Remove Pod from Service endpoints (no restart) | Waiting for database/cache connections, maintenance mode |
| **Liveness Probe** | Check if the application is still healthy | Continuously, after startup | Restart the unhealthy container | Detecting deadlocks, hangs, crashed applications |

---

## Probe Lifecycle

```text
Container Starts
       │
       ▼
Startup Probe
       │
       ├── Fail → Keep waiting
       │
       └── Success
                │
                ▼
     Startup Probe Stops
                │
                ├───────────────┐
                ▼               ▼
      Readiness Probe    Liveness Probe
         (Traffic)          (Health)
```

---

## The Question Each Probe Answers

### Startup Probe
**"Have you finished starting?"**

- No → Wait.
- Yes → Stop the Startup Probe for good (for this container instance) and begin normal health monitoring.

### Readiness Probe
**"Can I send user traffic to you?"**

- No:
  ```text
  Running
  READY = 0/1
        │
        ▼
  Removed from Service
  ```
  No restart happens.

- Yes:
  ```text
  READY = 1/1
        │
        ▼
  Traffic Starts
  ```

### Liveness Probe
**"Are you still alive?"**

- Yes → Keep running.
- No:
  ```text
  Kill Container
        │
        ▼
  Restart Container
  ```

---

## A Simple Analogy: A New Employee Joining a Company

**Startup Probe**
> "Have you finished setting up your laptop, VPN, email, and software?"
> No → wait. Yes → this question is never asked again.

**Readiness Probe**
> "Can I assign customer tickets to you?"
> Yes → assign work. No → don't assign work (the employee is still employed either way).

**Liveness Probe**
> "Are you still working?"
> If the employee hangs or becomes unresponsive, they get replaced.

---

## Failure Behavior

| Probe Fails | What Happens? |
|---|---|
| Startup Probe | Waits until `failureThreshold` is reached; then restarts the container |
| Readiness Probe | Pod is removed from Service endpoints; container keeps running |
| Liveness Probe | Container is restarted |

---

## When Does Each Probe Run?

| Probe | Runs Once? | Runs Continuously? |
|---|---|---|
| Startup |  Yes (once per container startup) |  No |
| Readiness |  No |  Yes |
| Liveness |  No |  Yes |

---

## Full Timeline Example

```text
0 sec
Container Starts
      │
      ▼
Startup Probe

      │
      ▼
Application Finished Starting
      │
      ▼
Startup Probe 
      │
      ▼
Startup Probe Stops Forever
(for this container)
      │
      ▼
Readiness Probe 
      │
      ▼
Traffic Begins
      │
      ▼
Liveness Probe    
      │
      ▼
Application Hangs
      │
      ▼
Liveness Probe  
      │
      ▼
Container Restarted
      │
      ▼
New Container Starts
      │
      ▼
Startup Probe Runs Again
```

---

## Memory Trick

| Probe | Remember This Question |
|---|---|
| **Startup Probe** | Have I finished starting? |
| **Readiness Probe** | Can I receive traffic? |
| **Liveness Probe** | Am I still healthy? |

---

## One-Line Summary

> **Startup Probe** protects slow-starting applications, **Readiness Probe** controls when traffic is sent to a Pod, and **Liveness Probe** keeps the application healthy by restarting it if it becomes unresponsive.

This single sentence captures the purpose of all three probes — worth remembering for interviews as well as real-world Kubernetes deployments.
