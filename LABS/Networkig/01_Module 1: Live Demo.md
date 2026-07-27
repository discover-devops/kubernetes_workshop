# Module 1: Live Demo Script for Teaching Session
## Quick Reference + Executable Commands

---

## Timing Breakdown (15 mins total)

- **0-2 min:** Visual walkthrough (slides/diagram)
- **2-8 min:** Concept explanation with real examples
- **8-13 min:** Live lab demonstrations
- **13-15 min:** Q&A and key takeaways

---

##  Part 1: Concept Walkthrough (2 min)

### Script for You:

> "Before Kubernetes, applications lived on fixed servers with known IPs. When containers arrived, things got complex. Now, imagine you're running 300 containers across 10 nodes. Each container has an IP. But here's the problem: containers die and respawn constantly with NEW IPs.
>
> Think of it like this: Your app has a phone number (pod IP). But every time the app crashes, it gets a new phone number. Would you trust hardcoding phone numbers? No! 
>
> Kubernetes Networking solves this with a magic layer called **Services**. A Service is like having a permanent phone number that always routes to the latest app instance, no matter how many times it crashes and restarts.
>
> That's what we're going to explore today."

---

##  Part 2: Live Labs (Setup Before Session)

### Prerequisites (Run These Before Your Session)

```bash
# 1. Create a demo namespace
kubectl create namespace demo-networking --dry-run=client -o yaml | kubectl apply -f -

# 2. Verify connectivity to your EKS cluster
kubectl cluster-info
kubectl get nodes -w

# 3. Create initial test deployment
kubectl -n demo-networking apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: demo-networking
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.14-alpine
        ports:
        - containerPort: 80
EOF

# Wait for pods to be running
kubectl -n demo-networking get pods -w
# Should see 2 pods running after ~30 seconds
```

---

##  Lab 1: Pod IP Instability (5 minutes)

### Setup
```bash
# Terminal setup (you'll use 2 terminals for this)

# Terminal 1: Watch pods in real-time
kubectl -n demo-networking get pods -w -o wide

# Terminal 2: Will run commands to delete and recreate
# (prepare but don't run yet)
```

### Live Demo Steps (narrate as you do this)

```bash
# STEP 1: Show current pod IPs
echo "=== CURRENT PODS AND THEIR IPS ==="
kubectl -n demo-networking get pods -o wide

# Record the IPs:
# POD_A_NAME=$(kubectl -n demo-networking get pods -o jsonpath='{.items[0].metadata.name}')
# POD_A_IP=$(kubectl -n demo-networking get pods -o jsonpath='{.items[0].status.podIP}')
# POD_B_NAME=$(kubectl -n demo-networking get pods -o jsonpath='{.items[1].metadata.name}')
# POD_B_IP=$(kubectl -n demo-networking get pods -o jsonpath='{.items[1].status.podIP}')

# STEP 2: Create a debug pod to monitor connectivity
echo "=== CREATING DEBUG POD ==="
kubectl -n demo-networking run debug-pod --image=curlimages/curl --rm -it -- sh

# STEP 3: Inside debug pod, test connectivity to a pod IP
# (Run these commands INSIDE the debug pod shell):
POD_IP="10.X.X.X"  # Use the actual IP from STEP 1
echo "Testing connectivity to pod at $POD_IP"

for i in {1..5}; do
  echo "Attempt $i:"
  curl -m 2 http://$POD_IP:80 > /dev/null 2>&1 && echo "✓ SUCCESS" || echo "✗ FAILED"
  sleep 1
done

# STEP 4: In another terminal, delete the pod
# (Terminal 2, while debug pod is running):
echo "=== DELETING A POD ==="
kubectl -n demo-networking delete pod $POD_A_NAME --wait=false

# STEP 5: Watch Terminal 1 - you'll see:
# - Old pod terminating
# - New pod being created with DIFFERENT IP
# - This takes ~5 seconds

# STEP 6: Back in debug pod shell, continue the curl loop
# You'll see the failures start happening!
# Even though K8s immediately created a replacement pod,
# that replacement pod has a DIFFERENT IP address!
```

### What Students See:

```
Terminal 1 (watching pods):
NAME                 READY   STATUS              IP          NODE
demo-app-abc-xyz     1/1     Running             10.0.23.4   node-1
demo-app-def-uvw     1/1     Running             10.0.45.8   node-2

[You delete the first pod...]

demo-app-abc-xyz     1/1     Terminating         10.0.23.4   node-1
demo-app-new-111     0/1     ContainerCreating               node-1
demo-app-new-111     1/1     Running             10.0.67.9   node-1
                                                 ↑ NEW IP!

Terminal 2 (in debug pod):
Attempt 1: curl to 10.0.23.4:80 ✓ SUCCESS
Attempt 2: curl to 10.0.23.4:80 ✓ SUCCESS
[Pod deleted]
Attempt 3: curl to 10.0.23.4:80 ✗ FAILED (pod is gone!)
Attempt 4: curl to 10.0.23.4:80 ✗ FAILED
Attempt 5: curl to 10.0.23.4:80 ✗ FAILED

KEY INSIGHT:
The new pod is running at 10.0.67.9, but we're still 
trying to call 10.0.23.4. That IP is DEAD FOREVER.
```

### What You Say:

> "See what just happened? We had a pod at 10.0.23.4. It crashed. K8s created a replacement immediately. But that new pod has a completely different IP: 10.0.67.9.
>
> If your frontend code was hardcoded to call 10.0.23.4, it would break the moment that pod died. Even though a replacement exists, the app never finds it.
>
> **This is the core problem Kubernetes Networking solves.**"

---

##  Lab 2: Cross-Node Communication (3 minutes)

### Setup (Before Session)

```bash
# Create a deployment that spreads across multiple nodes
kubectl -n demo-networking apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-node-test
  namespace: demo-networking
spec:
  replicas: 3
  selector:
    matchLabels:
      app: multi-node-test
  template:
    metadata:
      labels:
        app: multi-node-test
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - multi-node-test
              topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:1.14-alpine
        ports:
        - containerPort: 80
EOF

# Wait for pods to spread across nodes
kubectl -n demo-networking get pods -w -o wide
# Should see pods on different nodes
```

### Live Demo Steps

```bash
# STEP 1: Show pod distribution
echo "=== PODS DISTRIBUTED ACROSS NODES ==="
kubectl -n demo-networking get pods -o wide
# NAME                               IP           NODE
# multi-node-test-abc-1              10.0.23.4    node-1
# multi-node-test-abc-2              10.0.45.8    node-2  ← Different node!
# multi-node-test-abc-3              10.0.67.9    node-3  ← Different node!

# STEP 2: Test cross-node connectivity
echo "=== TESTING CROSS-NODE COMMUNICATION ==="
POD1=$(kubectl -n demo-networking get pods -o jsonpath='{.items[0].metadata.name}')
POD3_IP=$(kubectl -n demo-networking get pods -o jsonpath='{.items[2].status.podIP}')

echo "Pod 1 on node-1 is calling Pod 3 on node-3..."
kubectl -n demo-networking exec $POD1 -- curl -s http://$POD3_IP:80 | head -5

# You'll get HTML response from Pod 3's nginx!
```

### What You Say:

> "Here's something remarkable: Pod 1 is running on node-1. Pod 3 is running on node-3. They're on completely different machines. Yet Pod 1 can reach Pod 3 directly as if they're on the same local network.
>
> This is the **first golden rule of Kubernetes networking**: All pods can communicate with all other pods, no matter which node they're on. No special routing configuration needed. No NAT tricks. Transparent, flat networking.
>
> Behind the scenes, there's a lot of magic happening—CNI plugins, veth pairs, overlay networks, routing. But to the application, it's seamless.
>
> This is only possible because K8s has solved the networking problem at the infrastructure level."

---

##  Part 3: Concept Wrap-Up (2 minutes)

### The Two Problems Stated

**Problem 1: Ephemeral Pods**
```
Application needs stable access point.
Pods have temporary IPs.
Solution: Service = stable virtual IP
```

**Problem 2: Dynamic Discovery**
```
Don't know which pod IPs exist.
Pods appear and disappear.
Solution: Service = auto-discovers healthy pods
```

### Quick Q&A to Ask Students:

1. **"Why can't we use pod IPs in our code?"**
   - Expected answer: "Because they change when pods restart"
   - Your confirmation: "Exactly. Even though K8s replaces pods instantly, new pods get new IPs. Hardcoding breaks."

2. **"How does a pod on node-1 send data to a pod on node-3?"**
   - Expected answer: Might mention routing, plugins, etc.
   - Your confirmation: "Great! K8s uses CNI plugins to create an overlay network that makes all pods appear on the same flat network, hiding the complexity."

3. **"What if I could have a permanent IP that always routes to the 'web-app', regardless of which pods are currently running?"**
   - Expected answer: Might say "That would solve it!" or "That's a Service!"
   - Your confirmation: "Exactly. That's precisely what a Kubernetes Service provides."

---

##  Key Takeaways (Final 1 minute)

Print or display this:

```
┌─────────────────────────────────────────────────────────────┐
│  MODULE 1 TAKEAWAYS: Why K8s Networking Matters            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✗ PROBLEM 1: Pod IPs are ephemeral                        │
│    → Pods restart with new IPs                            │
│    → Hardcoding IPs breaks production                      │
│                                                             │
│  ✗ PROBLEM 2: Multi-node complexity                        │
│    → Need seamless cross-node communication               │
│    → Can't hardcode node routing                          │
│                                                             │
│  ✗ PROBLEM 3: Dynamic pod inventory                        │
│    → Pods scale up/down constantly                         │
│    → No way to track healthy endpoints                     │
│                                                             │
│  ✓ SOLUTION: Kubernetes Services                           │
│    → Stable virtual IP (never changes)                    │
│    → DNS name (friendly access)                           │
│    → Auto load-balancing across pods                       │
│    → Auto-discovery of healthy pods                        │
│                                                             │
│  ✓ NEXT: Learn the 2 Golden Rules of K8s Networking      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

##  Cleanup (After Session)

```bash
# Keep the demo namespace for next module, or clean it up
kubectl delete namespace demo-networking  # Optional

# Verify cleanup
kubectl get namespaces | grep demo
```

---

## Troubleshooting Tips

### Problem: Pods not spreading across nodes
```bash
# Check node count
kubectl get nodes

# If only 1-2 nodes, anti-affinity won't work
# Just proceed—the concept still applies
```

### Problem: Curl inside pod doesn't work
```bash
# Debug: shell into pod and test
kubectl -n demo-networking exec -it <pod-name> -- sh
/ # curl http://10.x.x.x:80
```

### Problem: Deployment doesn't start
```bash
# Check pod logs
kubectl -n demo-networking logs <pod-name>

# Check events
kubectl -n demo-networking describe pod <pod-name>
```

### Problem: Pod IP not visible
```bash
# Add `-o wide` to see IPs
kubectl -n demo-networking get pods -o wide
```

---

##  Command Cheat Sheet

```bash
# Create namespace
kubectl create namespace demo-networking --dry-run=client -o yaml | kubectl apply -f -

# Create deployment
kubectl -n demo-networking apply -f deployment.yaml

# Watch pods
kubectl -n demo-networking get pods -w -o wide

# Get specific info
POD_NAME=$(kubectl -n demo-networking get pods -o jsonpath='{.items[0].metadata.name}')
POD_IP=$(kubectl -n demo-networking get pods -o jsonpath='{.items[0].status.podIP}')
echo $POD_NAME
echo $POD_IP

# Delete pod (triggers restart)
kubectl -n demo-networking delete pod <pod-name> --wait=false

# Run debug/test pod
kubectl -n demo-networking run debug-pod --image=curlimages/curl -it --rm -- sh

# Exec into pod
kubectl -n demo-networking exec -it <pod-name> -- sh

# View pod details
kubectl -n demo-networking describe pod <pod-name>

# Cleanup
kubectl delete namespace demo-networking
```

---

##  Talking Points Summary

| Topic | Key Point | Example |
|-------|-----------|---------|
| **Pod Ephemerality** | Pods are temporary | A pod crashes, new one spawns at different IP |
| **Fixed IPs Don't Work** | IPs change constantly | Can't hardcode 10.0.1.5 when pod might be 10.0.2.9 later |
| **Service Discovery** | Need to find active pods dynamically | 5 replicas of "web-app" might be at 5 different IPs |
| **Cross-Node** | Nodes appear isolated but aren't | Flat networking makes multi-node transparent |
| **The Solution** | Services solve all these problems | Service = stable IP + load balancer + auto-discovery |

---

##  Pre-Session Checklist

- [ ] EKS cluster is running and accessible
- [ ] Have kubectl configured and working
- [ ] Created demo-networking namespace
- [ ] Tested deployments create and scale properly
- [ ] Know your cluster's node count (for multi-node demo)
- [ ] Have 2 terminal windows open (one for watching, one for commands)
- [ ] Test curl/networking tools work in container
- [ ] Review the visual diagram one more time
- [ ] Know your timing (15 mins is tight!)

---

##  Quick Run-Through (Do This 5 Minutes Before Session)

```bash
# Just run these to verify everything works:

# 1. Verify cluster
kubectl cluster-info | head -1

# 2. Check nodes
kubectl get nodes -o wide

# 3. Check namespace exists
kubectl get namespace demo-networking

# 4. Check deployments are running  
kubectl -n demo-networking get pods

# 5. Verify one pod is reachable
POD_IP=$(kubectl -n demo-networking get pods -o jsonpath='{.items[0].status.podIP}')
kubectl -n demo-networking run test --image=curlimages/curl --rm -it -- curl -m 2 http://$POD_IP:80 > /dev/null 2>&1 && echo "✓ Everything works!" || echo "✗ Issue found"
```

All green? You're ready to teach!
