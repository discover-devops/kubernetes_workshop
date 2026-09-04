# Kubernetes Troubleshooting Story: Calico CNI Installation Failure

## Interview Scenario

**Interviewer:** *Tell me about a Kubernetes issue you faced while setting up a cluster and how you troubleshot it.*

### Candidate Answer

> While setting up a Kubernetes cluster using `kubeadm`, I created a three-node cluster with one control-plane node and two worker nodes.
>
> Initially, the control-plane node was working and the worker nodes were able to join the cluster. When I checked the nodes using:
>
> ```bash
> kubectl get nodes
> ```
>
> all three nodes were showing as `Ready`.
>
> However, when I checked the system Pods:
>
> ```bash
> kubectl get pods -n kube-system
> ```
>
> I noticed that the Calico Pods were not becoming ready. They were showing `0/1`, even though their status was `Running`.

---

## Step 1 — Don't Assume That `Running` Means Healthy

The first thing I did was inspect the problematic Pod instead of immediately reinstalling Kubernetes.

I checked the Pod details and events:

```bash
kubectl describe pod -n kube-system calico-kube-controllers-<pod-name>
```

The events showed:

```text
Liveness probe failed:
Error: unknown shorthand flag: 'l' in -l

Readiness probe failed:
Error: unknown shorthand flag: 'r' in -r
```

I also noticed that the Calico image being used was:

```text
quay.io/calico/calico:master
```

At this point, I understood that this wasn't simply a case of "Calico needs more time."

**The Calico component itself was unhealthy.**

---

# Step 2 — Check the Rest of Calico

I then checked all Calico Pods:

```bash
kubectl get pods -n kube-system -o wide
```

I found:

```text
calico-kube-controllers    0/1
calico-node                0/1
calico-node                0/1
calico-node                0/1
```

So the problem wasn't isolated to a single Pod.

I also checked the DaemonSet:

```bash
kubectl get ds -n kube-system
```

and found:

```text
calico-node   DESIRED 3   CURRENT 3   READY 0
```

This confirmed that Calico had been deployed to all three nodes, but **none of the Calico nodes were actually becoming Ready**.

---

# Step 3 — Remove the Broken Calico Installation

Since this was a fresh lab cluster and we had identified that the initial Calico installation was problematic, I decided not to manually modify the health probes.

I removed the existing Calico installation.

During the cleanup, I noticed that the Calico controller Pod became:

```text
Terminating
```

but it wasn't actually disappearing.

I verified it with:

```bash
kubectl get pods -n kube-system
```

The Pod remained stuck in:

```text
0/1   Terminating
```

---

# Step 4 — Handle the Stuck Pod

Because the Pod was stuck in the terminating state and this was a disposable lab environment, I forcefully deleted that specific Pod:

```bash
kubectl delete pod <pod-name> \
  -n kube-system \
  --grace-period=0 \
  --force
```

After that, I verified that the old Calico Pod was gone:

```bash
kubectl get pods -A | grep -i calico
```

At this point, the old Calico Pods were removed.

---

# Step 5 — Reinstall Calico

I then installed the stable Calico version specified for the lab.

I applied the Calico CRDs:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.2/manifests/v1_crd_projectcalico_org.yaml
```

and the Tigera Operator:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.2/manifests/tigera-operator.yaml
```

However, Kubernetes returned several:

```text
AlreadyExists
```

errors.

Instead of treating those as a new failure, I recognized that these resources were **already present**.

For example:

```text
customresourcedefinitions ... already exists
namespace "tigera-operator" already exists
deployment "tigera-operator" already exists
```

I checked the operator:

```bash
kubectl get pods -n tigera-operator
```

and found:

```text
tigera-operator   1/1   Running
```

So the operator itself was healthy.

---

# Step 6 — The Real Root Cause Appeared

I then checked the Calico status:

```bash
kubectl get tigerastatus
```

This gave me the most important clue:

```text
kubeadm configuration is missing required podSubnet field
```

That was the breakthrough.

I then checked whether the nodes had Pod CIDRs assigned:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.podCIDR}{"\n"}{end}'
```

The result was:

```text
k8s-master
k8s-worker-01
k8s-worker-02
```

There was **no Pod CIDR assigned to any node**.

So we had moved from:

> "Calico is broken"

to a much more precise root cause:

> **The kubeadm cluster had been initialized without the Pod network CIDR required by our Calico setup.**

---

# Step 7 — Rebuild the Fresh Lab Cluster Correctly

Because this was a fresh training environment, rather than trying to patch the networking configuration of an already partially configured cluster, I chose the cleaner approach:

**Reset and initialize the cluster correctly.**

On the control plane:

```bash
sudo kubeadm reset -f
```

I also removed the old CNI configuration:

```bash
sudo rm -rf /etc/cni/net.d
```

Then I initialized the control plane again with the Pod network CIDR:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=10.0.1.172 \
  --pod-network-cidr=192.168.0.0/16
```

The important difference was:

```text
--pod-network-cidr=192.168.0.0/16
```

---

# Step 8 — Reconnect the Workers

Because we had recreated the control plane, the previous worker join information was no longer valid.

I generated a new join command:

```bash
kubeadm token create --print-join-command
```

This produced a new:

```bash
kubeadm join ...
```

command.

When I initially tried to join Worker 01, Kubernetes returned:

```text
/etc/kubernetes/kubelet.conf already exists
Port 10250 is in use
/etc/kubernetes/pki/ca.crt already exists
```

That indicated that the worker still contained configuration from the **old cluster**.

So I reset the worker:

```bash
sudo kubeadm reset -f
```

and cleaned the old CNI configuration:

```bash
sudo rm -rf /etc/cni/net.d
```

Then I ran the new:

```bash
sudo kubeadm join ...
```

I repeated the same cleanup and join process for Worker 02.

---

# Step 9 — Final Verification

Finally, from the control plane:

```bash
kubectl get nodes
```

All three nodes were healthy:

```text
k8s-master       Ready
k8s-worker-01    Ready
k8s-worker-02    Ready
```

Then:

```bash
kubectl get pods -A
```

showed the Calico components as healthy:

```text
calico-apiserver             1/1   Running
calico-kube-controllers      1/1   Running
calico-node                  1/1   Running
calico-typha                 1/1   Running
```

CoreDNS was also healthy:

```text
coredns                       1/1   Running
coredns                       1/1   Running
```

So the final architecture was working correctly:

**kubeadm → Pod CIDR configured → Calico CNI → Pod networking → CoreDNS → healthy Kubernetes cluster**

---

# What I Would Say as the Root Cause

If the interviewer asks:

### "What was the root cause?"

A strong answer is:

> The initial Calico installation was unhealthy, and during troubleshooting we discovered that the kubeadm cluster had been initialized without a Pod network CIDR. The nodes therefore had no Pod CIDRs assigned, which prevented the Calico operator from creating the required IP pool correctly. Since this was a fresh lab environment, we reset the cluster and reinitialized kubeadm with `--pod-network-cidr=192.168.0.0/16`, then reinstalled Calico and rejoined the workers.

---

# Interview Questions They May Ask

## 1. Why did you check `kubectl describe pod`?

**Answer:**

> Because `kubectl get pods` only tells me the current state. `kubectl describe pod` gives me events, probe failures, scheduling information, image details, and other clues about why the Pod isn't becoming Ready.

---

## 2. What is the difference between `Running` and `Ready`?

**Answer:**

> `Running` means the container has started. `Ready` means the Pod has successfully passed its readiness check and is considered ready to receive traffic or perform its intended function.

In our case:

```text
Running + 0/1 Ready
```

meant the container was running but failing its health checks.

---

## 3. Why did you check the DaemonSet?

**Answer:**

> Calico Node runs as a DaemonSet because we need a Calico networking agent on each Kubernetes node. Checking the DaemonSet allowed me to verify whether Calico had been scheduled on all nodes and whether those instances were actually Ready.

---

## 4. Why did you check `tigerastatus`?

**Answer:**

> Because we were using the Tigera Operator. `tigerastatus` gives us the operator's view of the health and progress of the Calico components and helped expose the missing `podSubnet` configuration.

---

## 5. What does `--pod-network-cidr` do?

**Answer:**

> It defines the IP address range that Kubernetes will use for Pod networking. The CNI uses this network to assign IP addresses to Pods and establish communication between Pods across nodes.

---

## 6. Why did you use `192.168.0.0/16`?

**Answer:**

> We selected `192.168.0.0/16` as the Pod network CIDR for this lab so that the Kubernetes Pod network would be separate from the EC2 node network, which was using the `10.0.1.x` range.

---

## 7. Why didn't you just edit the Calico configuration?

**Answer:**

> Since this was a newly created lab cluster and the cluster had been initialized incorrectly, rebuilding it was cleaner and less error-prone than trying to repair partially configured networking components. In a production environment, I would first evaluate the impact and look for a non-disruptive remediation.

---

## 8. Why did the old worker fail to join?

The worker reported:

```text
/etc/kubernetes/kubelet.conf already exists
Port 10250 is in use
/etc/kubernetes/pki/ca.crt already exists
```

**Answer:**

> The worker still had state from the previous Kubernetes cluster. `kubeadm reset` removed the old cluster configuration, after which the worker could join the newly initialized control plane.

---

## 9. Why did you generate a new join command?

**Answer:**

> Because we ran `kubeadm reset` and initialized the control plane again. The new cluster generated new bootstrap credentials, so I generated a fresh join command using `kubeadm token create --print-join-command`.

---

## 10. Why didn't you use the old join command?

**Answer:**

> The old join command belonged to the previous cluster state. After recreating the control plane, I used a newly generated token and discovery hash to make sure the workers joined the current cluster.

---

# The Interview Troubleshooting Framework

The biggest takeaway for students is **not the individual commands**.

Remember this flow:

```text
Problem
   ↓
Observe
   ↓
kubectl get pods
   ↓
Pod is Running but Not Ready
   ↓
kubectl describe pod
   ↓
Health probe errors
   ↓
Check Calico components
   ↓
Check DaemonSet
   ↓
Check Tigera status
   ↓
Find missing podSubnet
   ↓
Check node Pod CIDRs
   ↓
Pod CIDRs are missing
   ↓
Root cause identified
   ↓
Reset fresh lab cluster
   ↓
kubeadm init + --pod-network-cidr
   ↓
Install Calico
   ↓
Reset stale workers
   ↓
kubeadm join
   ↓
Verify
   ↓
All nodes + Calico + CoreDNS Ready
```

### The interview mindset

Don't say:

> "Calico wasn't working, so I reinstalled Calico."

Say:

> **"I started from the symptom, collected evidence using Pod events and Calico status, identified that the nodes had no Pod CIDRs, traced that back to the kubeadm initialization, and rebuilt the fresh cluster with the correct Pod network configuration."**

That second answer demonstrates **actual troubleshooting methodology**, which is what the interviewer is really testing.
