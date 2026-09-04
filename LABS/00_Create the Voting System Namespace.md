
# Step 1 — Create the Voting System Namespace

## What are we building?

Our application will contain multiple Kubernetes resources:

```text
Blue Team
Red Team
Redis
Results Dashboard
```

Instead of putting everything into the default namespace, we will create a dedicated namespace:

```text
voting-system
```

Our application architecture will eventually look like:

```text
                    Kubernetes Cluster
                           |
                    voting-system
                           |
          +----------------+----------------+
          |                |                |
      Blue Team         Red Team          Redis
          |                |                |
      NodePort           NodePort        ClusterIP
      :30001             :30002            :6379
          |                |                |
          +----------------+----------------+
                           |
                    Results Dashboard
                         :30003
```

---

# 1.1 What is a Namespace?

A **Namespace** is a logical boundary inside a Kubernetes cluster.

Think of the cluster as a large building:

```text
Kubernetes Cluster
│
├── Namespace: voting-system
│   ├── Blue Pods
│   ├── Red Pods
│   ├── Redis
│   └── Results Dashboard
│
├── Namespace: kube-system
│   ├── CoreDNS
│   ├── kube-proxy
│   └── other system components
│
└── Namespace: default
```

We don't want our application resources mixed together with Kubernetes system resources.

### Why use a Namespace?

It helps us:

* organize resources
* isolate applications
* manage permissions
* apply resource quotas
* make commands easier to scope

For this lab, all our application resources will live inside:

```text
voting-system
```

---

# 1.2 Create the Namespace

On the **master/control-plane node**, create a file:

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

Save and exit.

If using `nano`:

```text
Ctrl + O
Enter
Ctrl + X
```

---

# 1.3 Understand the YAML

Let's understand each line.

### `apiVersion`

```yaml
apiVersion: v1
```

This tells Kubernetes which API version should be used for this resource.

For a Namespace, we use:

```text
v1
```

---

### `kind`

```yaml
kind: Namespace
```

This tells Kubernetes:

> "I want to create a Namespace."

---

### `metadata`

```yaml
metadata:
  name: voting-system
```

Metadata contains information that identifies the resource.

Here, the Namespace name is:

```text
voting-system
```

---

# 1.4 Create the Namespace

Run:

```bash
kubectl apply -f 01-namespace.yaml
```

Expected:

```text
namespace/voting-system created
```

---

# 1.5 Verify

Run:

```bash
kubectl get namespaces
```

You should see:

```text
NAME              STATUS   AGE
default           Active   ...
kube-node-lease   Active   ...
kube-public       Active   ...
kube-system       Active   ...
voting-system     Active   ...
```

You can also run:

```bash
kubectl get ns voting-system
```

Expected:

```text
NAME            STATUS   AGE
voting-system   Active   ...
```

---

# 1.6 Important Concept — `-n`

From now on, most application resources will be created inside:

```text
voting-system
```

So you'll frequently see:

```bash
kubectl get pods -n voting-system
```

The `-n` means:

```text
--namespace
```

Therefore:

```bash
kubectl get pods -n voting-system
```

means:

> "Show me the Pods belonging to the `voting-system` namespace."

You can also write:

```bash
kubectl get pods --namespace=voting-system
```

Both are equivalent.

---

# 1.7 Why not use the `default` namespace?

Students often ask this.

You *can* deploy applications into `default`, but in a real environment it becomes difficult to manage a cluster when everything is mixed together.

For example:

```text
default
├── application-a
├── application-b
├── redis
├── nginx
├── test-pod
├── monitoring
└── random-debug-pod
```

Using namespaces gives us a cleaner structure:

```text
voting-system
├── blue
├── red
├── redis
└── results
```

while Kubernetes itself continues using:

```text
kube-system
```

for system components.

---

# 1.8 Useful Namespace Commands

List namespaces:

```bash
kubectl get ns
```

Get resources inside our namespace:

```bash
kubectl get all -n voting-system
```

At this point it will probably show:

```text
No resources found
```

That's expected because we have created only the Namespace.

Describe the Namespace:

```bash
kubectl describe namespace voting-system
```

---

# 1.9 Troubleshooting

### Error: Namespace already exists

If you run:

```bash
kubectl apply -f 01-namespace.yaml
```

and Kubernetes says the namespace already exists, that's usually not a problem.

Check:

```bash
kubectl get ns voting-system
```

If you see:

```text
voting-system   Active
```

you're good.

---

# 1.10 Verify Our Starting Point

Before moving to the Blue Team application, run:

```bash
kubectl get nodes
```

You should have:

```text
k8s-master       Ready
k8s-worker-01    Ready
k8s-worker-02    Ready
```

Check Calico:

```bash
kubectl get pods -n calico-system
```

Calico Pods should be `Running` and Ready.

Check the application namespace:

```bash
kubectl get all -n voting-system
```

Currently, there should be no application Pods.

---

# What We Have Achieved

We now have:

```text
Kubernetes Cluster
        |
        +── kube-system
        |     ├── CoreDNS
        |     ├── kube-proxy
        |     └── Kubernetes components
        |
        +── calico-system
        |     └── Calico networking
        |
        +── voting-system
              └── Empty application namespace
```

The namespace gives us a clean home for the Voting Application.

---

# Next — Blue Team Application

Now we'll create our first actual workload.

The flow will be:

```text
Blue ConfigMap
      ↓
Blue Deployment
      ↓
Blue Pods
      ↓
Blue Service
      ↓
NodePort :30001
      ↓
Browser
      ↓
EC2 Public IP
```

This will introduce the most important Kubernetes concepts together:

* **ConfigMap**
* **Deployment**
* **Pod**
* **Labels**
* **Selectors**
* **Service**
* **NodePort**
* **RollingUpdate**
* **Liveness probe**
* **Readiness probe**
* **Resource requests/limits**
* **Pod anti-affinity**

That is the next step in the lab.
