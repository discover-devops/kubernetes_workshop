
# VoteVibe Kubernetes Lab — Minikube

## Student Runbook

### What will we build?

By the end of this lab:

```text
                         Minikube
                            |
                     voting-system
                            |
          +-----------------+-----------------+
          |                 |                 |
      Blue Team          Red Team           Redis
       :30001             :30002             :6379
          |                 |                 |
          +-----------------+-----------------+
                            |
                     Results Dashboard
                          :30003
```

We will learn:

```text
Namespace
ConfigMap
Deployment
Pod
Labels & Selectors
Service
NodePort
ClusterIP
DNS
Redis
Secret
PV / PVC
Probes
Metrics Server
HPA
Scaling
Load Testing
Rolling Update
Rollback
Troubleshooting
```

---

# Part 0 — Prerequisites

You need a Linux machine with:

* Ubuntu 22.04/24.04
* 2+ CPU
* 4 GB+ RAM
* 20 GB+ disk
* Internet access

We will use:

```text
Docker
Minikube
kubectl
```

---

# Step 1 — Install Docker

Update the machine:

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

Install Docker:

```bash
sudo apt-get install -y docker.io
```

Enable Docker:

```bash
sudo systemctl enable --now docker
```

Verify:

```bash
docker --version
```

Test:

```bash
sudo docker run hello-world
```

You should see the Docker welcome message.

---

# Step 2 — Allow Your User to Run Docker

Run:

```bash
sudo usermod -aG docker $USER
```

Then:

```bash
newgrp docker
```

Verify:

```bash
docker ps
```

You should **not** need `sudo`.

---

# Step 3 — Install kubectl

Download the latest stable kubectl:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Install it:

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Verify:

```bash
kubectl version --client
```

---

# Step 4 — Install Minikube

Download Minikube:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```

Install:

```bash
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

Verify:

```bash
minikube version
```

---

# Step 5 — Start Kubernetes

We will use Docker as the Minikube driver:

```bash
minikube start --driver=docker
```

This creates a Kubernetes cluster inside Docker.

Check:

```bash
minikube status
```

You should see the major components as running.

---

# Step 6 — Verify Kubernetes

Run:

```bash
kubectl get nodes
```

Expected:

```text
NAME       STATUS   ROLES           AGE
minikube   Ready    control-plane   ...
```

Check system Pods:

```bash
kubectl get pods -A
```

CoreDNS and the Kubernetes system components should be running.

---

# Step 7 — Verify the CNI / Networking

Minikube manages the networking for us.

Unlike our EC2 kubeadm setup, **we do not install Calico separately for this Minikube runbook**.

This is intentional.

In the EC2 lab:

```text
EC2
 ↓
kubeadm
 ↓
Calico
```

In this Minikube lab:

```text
Minikube
 ↓
Minikube-managed Kubernetes networking
```

The application-level Kubernetes concepts remain the same.

---

# Step 8 — Create the Application Namespace

Create:

```bash
nano 01-namespace.yaml
```

Paste:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: voting-system
```

Apply:

```bash
kubectl apply -f 01-namespace.yaml
```

Verify:

```bash
kubectl get ns voting-system
```

Expected:

```text
NAME            STATUS   AGE
voting-system   Active   ...
```

---

# Step 9 — Blue Team ConfigMap

Create:

```bash
nano 02-blue-configmap.yaml
```

Paste:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: blue-team-config
  namespace: voting-system
data:
  TEAM_NAME: "Blue Team"
  TEAM_COLOR: "#1976d2"
  BACKGROUND_COLOR: "#e3f2fd"
```

Apply:

```bash
kubectl apply -f 02-blue-configmap.yaml
```

Verify:

```bash
kubectl get configmap -n voting-system
```

---

# Step 10 — Blue Team Deployment

Create:

```bash
nano 03-blue-deployment.yaml
```

Use the **same Blue Team Deployment manifest from the EC2 lab**.

The important parts remain:

```text
2 replicas
Python/Flask
ConfigMap
labels
selectors
resource requests/limits
liveness probe
readiness probe
rolling update
pod anti-affinity
```

Apply:

```bash
kubectl apply -f 03-blue-deployment.yaml
```

Check:

```bash
kubectl get deployment -n voting-system
```

Then:

```bash
kubectl get pods -n voting-system -o wide
```

Expected:

```text
blue-team-deployment-xxxxx   1/1   Running
blue-team-deployment-yyyyy   1/1   Running
```

---

# Step 11 — Blue Team Service

Create:

```bash
nano 04-blue-service.yaml
```

Paste:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: blue-team-service
  namespace: voting-system
spec:
  type: NodePort
  selector:
    app: blue-team
  ports:
    - port: 80
      targetPort: 5000
      nodePort: 30001
  sessionAffinity: ClientIP
```

Apply:

```bash
kubectl apply -f 04-blue-service.yaml
```

Verify:

```bash
kubectl get svc -n voting-system
```

You should see:

```text
blue-team-service   NodePort   ...   80:30001/TCP
```

---

# Step 12 — Access Blue Team

With Minikube, don't use the EC2 public-IP approach.

Run:

```bash
minikube service blue-team-service -n voting-system
```

Minikube will provide the accessible URL.

You can also get the URL without opening the browser:

```bash
minikube service blue-team-service -n voting-system --url
```

Copy the returned URL and open it.

You should see:

```text
Blue Team
VoteVibe
Pod: ...
Namespace: voting-system
```

---

# Step 13 — Understand the Blue Team Architecture

At this point:

```text
Browser
   |
   ↓
Minikube NodePort :30001
   |
   ↓
Blue Service :80
   |
   ↓
Blue Pods :5000
```

And:

```text
Deployment
    ↓
ReplicaSet
    ↓
2 Pods
```

The Service finds the Pods using:

```text
app=blue-team
```

This is the first important **label → selector** relationship.

---

# Step 14 — Inspect and Troubleshoot

List everything:

```bash
kubectl get all -n voting-system
```

Check Pods:

```bash
kubectl get pods -n voting-system -o wide
```

Describe a Pod:

```bash
kubectl describe pod -n voting-system <POD_NAME>
```

View logs:

```bash
kubectl logs -n voting-system <POD_NAME>
```

Check Service:

```bash
kubectl describe svc blue-team-service -n voting-system
```

Check endpoints:

```bash
kubectl get endpoints -n voting-system
```

You should see the Blue Pod IPs behind the Service.

---

# Step 15 — Redis

Now we introduce **service-to-service communication**.

Architecture:

```text
Red Team
    |
    | redis-service:6379
    ↓
Redis
```

Redis will use:

```text
ClusterIP
```

because Redis does not need to be exposed to the internet.

Create the Redis Deployment using the lab's Redis manifest.

```bash
kubectl apply -f 05-redis-deployment.yaml
```

Verify:

```bash
kubectl get deployment -n voting-system
kubectl get pods -n voting-system
```

---

# Step 16 — Redis Service

Create the ClusterIP Service:

```bash
kubectl apply -f 06-redis-service.yaml
```

Verify:

```bash
kubectl get svc -n voting-system
```

You should have:

```text
redis-service   ClusterIP   ...   6379/TCP
```

Notice:

```text
Blue Service     NodePort
Redis Service    ClusterIP
```

### Why?

Blue Team needs user access.

Redis only needs internal application access.

---

# Step 17 — Test Kubernetes DNS

Run:

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  -n voting-system \
  --rm -it \
  -- nslookup redis-service
```

You should get a DNS response.

Now test the full DNS name:

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  -n voting-system \
  --rm -it \
  -- nslookup redis-service.voting-system.svc.cluster.local
```

The Kubernetes DNS pattern is:

```text
<service>.<namespace>.svc.cluster.local
```

---

# Step 18 — Red Team

Create the Red Team ConfigMap:

```bash
kubectl apply -f 07-red-configmap.yaml
```

Create the Deployment:

```bash
kubectl apply -f 08-red-deployment.yaml
```

Create the Service:

```bash
kubectl apply -f 09-red-service.yaml
```

Verify:

```bash
kubectl get all -n voting-system
```

You should now have:

```text
Blue Team
Red Team
Redis
```

Access Red:

```bash
minikube service red-team-service -n voting-system --url
```

---

# Step 19 — Test Red → Redis

Find a Red Pod:

```bash
kubectl get pods -n voting-system
```

Then test Redis DNS from the Red Pod:

```bash
kubectl exec -it -n voting-system <RED_POD> -- \
  getent hosts redis-service
```

You can also test Redis using a temporary client:

```bash
kubectl run redis-test \
  --image=redis:7.2-alpine \
  --restart=Never \
  -n voting-system \
  --rm -it \
  -- redis-cli -h redis-service -p 6379 ping
```

Expected:

```text
PONG
```

This proves:

```text
Red Pod
   ↓
Kubernetes DNS
   ↓
redis-service
   ↓
ClusterIP
   ↓
Redis Pod
```

---

# Step 20 — Redis Secret

Now we introduce sensitive configuration.

Create:

```bash
kubectl apply -f 10-redis-secret.yaml
```

Verify:

```bash
kubectl get secret -n voting-system
```

Remember:

> Base64 is encoding, not encryption.

In production, we would normally consider mechanisms such as AWS Secrets Manager, Vault, or Sealed Secrets depending on the architecture.

---

# Step 21 — Persistent Storage

Now we make Redis persistent.

Apply:

```bash
kubectl apply -f 11-redis-pv.yaml
```

Then:

```bash
kubectl apply -f 12-redis-pvc.yaml
```

Check:

```bash
kubectl get pv
```

and:

```bash
kubectl get pvc -n voting-system
```

Expected:

```text
redis-pvc   Bound
```

---

# Important Minikube Difference

In the EC2 lab, the Redis PV uses:

```text
hostPath
/data/redis-votevibe
```

and is tied to a particular worker node.

That is meaningful in our multi-node kubeadm environment.

Minikube is normally a **single-node cluster**, so node-local storage behaves differently.

For the Minikube student lab, we should use Minikube's local storage provisioner rather than teaching the EC2-specific nodeSelector/hostPath behavior as if it were production-grade storage.

---

# Step 22 — Redis v2

Deploy the persistent Redis version:

```bash
kubectl apply -f 13-redis-deployment-v2.yaml
```

Check:

```bash
kubectl get pods -n voting-system
```

Check PVC:

```bash
kubectl get pvc -n voting-system
```

We want:

```text
STATUS: Bound
```

---

# Step 23 — Results Dashboard

Deploy the dashboard configuration:

```bash
kubectl apply -f 14-results-configmap.yaml
```

Deploy the dashboard:

```bash
kubectl apply -f 15-results-deployment.yaml
```

Expose it:

```bash
kubectl apply -f 16-results-service.yaml
```

Verify:

```bash
kubectl get pods -n voting-system
kubectl get svc -n voting-system
```

Access:

```bash
minikube service results-service -n voting-system --url
```

The architecture is now:

```text
Blue ───────┐
            │
Red ────────┼──→ Redis
            │
Results ────┘
```

---

# Step 24 — Install Metrics Server

HPA requires resource metrics.

Check whether Metrics Server is already enabled:

```bash
minikube addons list | grep metrics
```

Enable it:

```bash
minikube addons enable metrics-server
```

Wait and verify:

```bash
kubectl get pods -n kube-system | grep metrics
```

Then:

```bash
kubectl top nodes
```

and:

```bash
kubectl top pods -n voting-system
```

You should eventually see CPU and memory usage.

---

# Step 25 — HPA

Apply the Blue HPA:

```bash
kubectl apply -f 17-hpa-blue.yaml
```

Apply the Red HPA:

```bash
kubectl apply -f 18-hpa-red.yaml
```

Check:

```bash
kubectl get hpa -n voting-system
```

You should see Blue and Red HPAs.

---

# Step 26 — Manual Scaling

Before testing automatic scaling, manually scale Blue Team:

```bash
kubectl scale deployment/blue-team-deployment \
  --replicas=4 \
  -n voting-system
```

Check:

```bash
kubectl get pods -n voting-system
```

Then return it to two:

```bash
kubectl scale deployment/blue-team-deployment \
  --replicas=2 \
  -n voting-system
```

---

# Step 27 — Load Test

Create traffic against Blue Team:

```bash
kubectl run load-gen \
  --image=busybox:1.36 \
  --restart=Never \
  -n voting-system \
  --rm -it \
  -- sh -c 'while true; do wget -q -O- http://blue-team-service/vote; done'
```

In another terminal:

```bash
kubectl get hpa -n voting-system -w
```

And:

```bash
kubectl get pods -n voting-system -w
```

You can observe Kubernetes responding to increased resource consumption.

Stop the load generator with:

```text
Ctrl+C
```

---

# Step 28 — Rolling Update

Now we will deliberately change the Blue Team configuration.

Run:

```bash
kubectl set env deployment/blue-team-deployment \
  TEAM_NAME="Blue Team v2.0" \
  -n voting-system
```

Watch the rollout:

```bash
kubectl rollout status deployment/blue-team-deployment \
  -n voting-system
```

Check:

```bash
kubectl get pods -n voting-system
```

Because our Deployment uses:

```text
RollingUpdate
```

Kubernetes gradually replaces the old Pods.

---

# Step 29 — Check Deployment History

Run:

```bash
kubectl rollout history deployment/blue-team-deployment \
  -n voting-system
```

You should see multiple revisions.

---

# Step 30 — Rollback

Suppose the new version has a problem.

We can undo it:

```bash
kubectl rollout undo deployment/blue-team-deployment \
  -n voting-system
```

Check:

```bash
kubectl rollout status deployment/blue-team-deployment \
  -n voting-system
```

And:

```bash
kubectl rollout history deployment/blue-team-deployment \
  -n voting-system
```

This demonstrates:

```text
Version 1
   ↓
Rolling Update
   ↓
Version 2
   ↓
Problem
   ↓
Rollback
   ↓
Version 1
```

---

# Step 31 — Final Health Check

Run:

```bash
kubectl get nodes
```

```bash
kubectl get all -n voting-system
```

```bash
kubectl get configmaps -n voting-system
```

```bash
kubectl get secrets -n voting-system
```

```bash
kubectl get pv
```

```bash
kubectl get pvc -n voting-system
```

```bash
kubectl get hpa -n voting-system
```

```bash
kubectl top nodes
```

```bash
kubectl top pods -n voting-system
```

---

# Final Architecture

At the end of the lab:

```text
                         MINIKUBE
                            |
                     voting-system
                            |
          +-----------------+------------------+
          |                 |                  |
      Blue Team          Red Team            Results
      Deployment         Deployment         Deployment
          |                 |                  |
       Service            Service            Service
       NodePort           NodePort           NodePort
       :30001             :30002             :30003
          |                 |                  |
          |                 +--------+---------+
          |                          |
          |                      Redis Service
          |                      ClusterIP :6379
          |                          |
          |                        Redis
          |                          |
          |                         PVC
          |                          |
          |                         PV
          |
       HPA  ← Metrics Server → HPA
```

---

## What You Should Be Able to Explain After This Lab

By the end, you should be able to explain:

**Why Namespace?**

```text
Logical isolation
```

**Why Deployment?**

```text
Desired state + Replica management + Rolling updates
```

**Why Service?**

```text
Stable endpoint for dynamic Pods
```

**Why NodePort?**

```text
External access to the application
```

**Why ClusterIP?**

```text
Internal service-to-service communication
```

**How does Red find Redis?**

```text
redis-service
        ↓
Kubernetes DNS
        ↓
ClusterIP
        ↓
Redis Pod
```

**Why ConfigMap?**

```text
Separate non-sensitive configuration from application
```

**Why Secret?**

```text
Sensitive configuration
```

**Why PVC?**

```text
Request persistent storage independently from the Pod
```

**Why HPA?**

```text
Automatically adjust replicas based on resource utilization
```

**Why probes?**

```text
Liveness  → Is the application alive?
Readiness → Can the application receive traffic?
```

**Why RollingUpdate?**

```text
Replace old Pods gradually instead of taking the whole application down
```

**Why Rollback?**

```text
Recover quickly when a deployment introduces a problem
```

This Minikube version gives students the **same application and Kubernetes learning journey**, while avoiding the AWS-specific infrastructure and the kubeadm/Calico bootstrap complexity that belongs in the EC2 version.
