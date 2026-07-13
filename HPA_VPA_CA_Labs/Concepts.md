## CPU: millicores (`m`)

- `1000m` = 1 full vCPU core.
- `500m` = 0.5 vCPU = half a core. Your read is correct.
- `200m` = 0.2 vCPU = one-fifth of a core.

So in our `php-apache` manifest:
- **`requests.cpu: 200m`** → the scheduler guarantees this pod at least 0.2 vCPU, and this is also the **baseline HPA uses to calculate utilization %**.
- **`limits.cpu: 500m`** → the container is hard-capped at 0.5 vCPU. Even under heavy load, it physically cannot use more than this — the kernel throttles it via CFS quota.

This gap between request (200m) and limit (500m) matters for our demo: when we hammer it with load, actual usage will climb toward 500m, which relative to the 200m *request* is **250% utilization** — well above our HPA target of 50%, so it'll scale out aggressively. Good for a clear demo.

## Memory: Mi = Mebibytes

- `Mi` = mebibyte = 2^20 bytes ≈ 1.048576 MB (note: `Mi` ≠ `M` — `M` is decimal megabytes, `Mi` is binary. Kubernetes convention prefers `Mi`).
- **`requests.memory: 100Mi`** → guaranteed 100 MiB reserved for this pod.
- **`limits.memory: 200Mi`** → hard cap at 200 MiB. Unlike CPU, if the container exceeds this, it doesn't get throttled — it gets **OOMKilled** (Out Of Memory kill) and restarted.

## Your node capacity — let's check it directly rather than guess

`t3.medium` nominally has 2 vCPU / 4 GiB RAM, but Kubernetes reserves a chunk for the kubelet, OS, and system daemons, so *allocatable* is always less than raw instance spec. Let's pull the real numbers off your actual nodes:

```bash
kubectl describe nodes | grep -A 8 "Allocatable:"

$ kubectl describe nodes | grep -A 8 "Allocatable:"
Allocatable:
  cpu:                1930m
  ephemeral-storage:  18181869946
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3376680Ki
  pods:               17
System Info:
  Machine ID:                 ec25d9c5f7eb37396379a304bb934078
--
Allocatable:
  cpu:                1930m
  ephemeral-storage:  18181869946
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3376688Ki
  pods:               17
System Info:
  Machine ID:                 ec2f511a57bc07e4c7ce511a6053256b
[ec2-user@ip-172-31-6-213 ~]$


```

or more precisely, just the capacity/allocatable block per node:

```bash
kubectl get nodes -o custom-columns='NAME:.metadata.name,CPU-CAP:.status.capacity.cpu,MEM-CAP:.status.capacity.memory,CPU-ALLOC:.status.allocatable.cpu,MEM-ALLOC:.status.allocatable.memory'
[ec2-user@ip-172-31-6-213 ~]$ kubectl get nodes -o custom-columns='NAME:.metadata.name,CPU-CAP:.status.capacity.cpu,MEM-CAP:.status.capacity.memory,CPU-ALLOC:.status.allocatable.cpu,MEM-ALLOC:.status.allocatable.memory'
NAME                                            CPU-CAP   MEM-CAP     CPU-ALLOC   MEM-ALLOC
ip-192-168-50-103.ap-south-1.compute.internal   2         3931688Ki   1930m       3376680Ki
ip-192-168-69-180.ap-south-1.compute.internal   2         3931696Ki   1930m       3376688Ki
[ec2-user@ip-172-31-6-213 ~]$

```


This confirms exactly what I expected for `t3.medium`. Let's decode it:

## CPU

| | Value | Meaning |
|---|---|---|
| **Capacity** | `2` (2000m) | Raw vCPU count of a `t3.medium` |
| **Allocatable** | `1930m` | What's actually schedulable for pods |
| **Reserved** | `70m` | Held back for kubelet + OS system daemons |

So each node gives up a small CPU tax (~3.5%) to keep the node itself healthy. This is standard EKS kubelet reservation behavior, not something specific to your setup.

## Memory

| | Value | Meaning |
|---|---|---|
| **Capacity** | `3931688Ki` ≈ **3.75 GiB** | Raw RAM of `t3.medium` (AWS lists it as 4 GiB, minor rounding from Ki conversion) |
| **Allocatable** | `3376680Ki` ≈ **3.22 GiB** | What's actually schedulable |
| **Reserved** | ≈ **555Mi** | Held back for kubelet, OS, and eviction thresholds |

So across your **2-node cluster**, total *schedulable* capacity right now is roughly:
- **CPU: ~3860m** (~3.86 vCPU total)
- **Memory: ~6.6 GiB total**

## The number that matters most for Lab 3 later

Notice `pods: 17` in the allocatable block — that's the **max pod count per node**, capped by ENI/IP limits on `t3.medium` (AWS VPC CNI restricts pods-per-node based on instance type, not just CPU/memory). So across 2 nodes, we can schedule a **maximum of 34 pods total** before Kubernetes refuses to schedule more, *even if CPU/memory technically has room*. This will directly matter in Lab 3 when we try to trigger the Cluster Autoscaler by exhausting capacity — we may hit the pod-count ceiling before we hit CPU/memory ceiling, which is actually a good real-world lesson to demonstrate.
