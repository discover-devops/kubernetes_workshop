# Step 2 — Blue Team: ConfigMap

We have created the `voting-system` namespace. Now we'll deploy our **first application component: Blue Team**.

Before creating the Deployment, we will separate the application's **configuration** from its application definition using a Kubernetes **ConfigMap**.

---

## 2.1 What are we building?

Our Blue Team application is a small Python/Flask voting application.

The application needs some configuration such as:

* Team name
* Team color
* Background color
* Redis configuration later
* Other environment-specific settings

Instead of hard-coding these values inside the application image or Deployment, Kubernetes can store non-sensitive configuration in a **ConfigMap**.

The flow is:

```text
ConfigMap
    ↓
Environment Variables
    ↓
Blue Team Pod
    ↓
Flask Application
```

---

# 2.2 What is a ConfigMap?

A **ConfigMap** is a Kubernetes object used to store **non-sensitive configuration data**.

For example:

```text
TEAM_NAME=Blue Team
TEAM_COLOR=blue
BACKGROUND_COLOR=#e3f2fd
```

The Pod can consume these values as environment variables.

### Why use a ConfigMap?

Suppose tomorrow we want to change:

```text
Blue Team
```

to:

```text
Blue Team v2
```

We shouldn't have to rebuild the Docker image just to change a configuration value.

Instead:

```text
Application code
        +
Configuration
```

are kept separate.

---

# 2.3 ConfigMap vs Secret

This distinction is important.

### ConfigMap

Used for **non-sensitive** configuration.

Examples:

```text
TEAM_NAME
PORT
APPLICATION_MODE
LOG_LEVEL
```

### Secret

Used for sensitive values.

Examples:

```text
PASSWORD
API_KEY
TOKEN
```

We will use a **Secret for the Redis password later**.

Do not put passwords or credentials in a ConfigMap.

---

# 2.4 Create the Blue ConfigMap

On the master:

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

Save and exit.

---

# 2.5 Understand the YAML

### API version

```yaml
apiVersion: v1
```

ConfigMap is part of the core Kubernetes API.

---

### Resource type

```yaml
kind: ConfigMap
```

We're telling Kubernetes:

> Create a ConfigMap.

---

### Name

```yaml
metadata:
  name: blue-team-config
```

Our ConfigMap is called:

```text
blue-team-config
```

---

### Namespace

```yaml
namespace: voting-system
```

This is important.

The ConfigMap belongs to our application namespace:

```text
voting-system
```

---

### Configuration data

```yaml
data:
  TEAM_NAME: "Blue Team"
  TEAM_COLOR: "#1976d2"
  BACKGROUND_COLOR: "#e3f2fd"
```

These are key-value pairs.

Think of it as:

```text
KEY                  VALUE
--------------------------------
TEAM_NAME            Blue Team
TEAM_COLOR           #1976d2
BACKGROUND_COLOR     #e3f2fd
```

---

# 2.6 Create the ConfigMap

Run:

```bash
kubectl apply -f 02-blue-configmap.yaml
```

Expected:

```text
configmap/blue-team-config created
```

---

# 2.7 Verify

Run:

```bash
kubectl get configmaps -n voting-system
```

Expected:

```text
NAME               DATA   AGE
blue-team-config   3      ...
```

You can inspect it:

```bash
kubectl describe configmap blue-team-config -n voting-system
```

You should see:

```text
TEAM_NAME
TEAM_COLOR
BACKGROUND_COLOR
```

You can also see the complete YAML:

```bash
kubectl get configmap blue-team-config -n voting-system -o yaml
```

---

# 2.8 Important Concept — ConfigMap Does Nothing by Itself

Creating the ConfigMap **does not automatically give the application access to it**.

We still need to tell the Pod:

> "Take these values from this ConfigMap and make them available to the container."

We'll do that in the Deployment.

The eventual flow is:

```text
blue-team-config
       |
       | envFrom
       ↓
Blue Team Pod
       |
       +-- TEAM_NAME
       +-- TEAM_COLOR
       +-- BACKGROUND_COLOR
```

---

# 2.9 Verify the Namespace

Let's check everything we've created so far:

```bash
kubectl get all -n voting-system
```

There are no Pods or Services yet.

Check ConfigMaps:

```bash
kubectl get configmap -n voting-system
```

You should see:

```text
blue-team-config
```

---

# 2.10 Common Mistake

### Mistake: Creating the ConfigMap in the wrong namespace

For example:

```bash
kubectl create configmap blue-team-config
```

without specifying the namespace would create it in the current/default namespace.

Our Deployment will be in:

```text
voting-system
```

and therefore it needs the ConfigMap in the same namespace.

That's why our YAML explicitly contains:

```yaml
namespace: voting-system
```

---

# What We Have Now

Our application currently looks like:

```text
Kubernetes Cluster
        |
        +── voting-system
              |
              +── blue-team-config
```

The ConfigMap contains:

```text
TEAM_NAME=Blue Team
TEAM_COLOR=#1976d2
BACKGROUND_COLOR=#e3f2fd
```

But we don't have a running application yet.

---

# Next Step — Blue Team Deployment

Now we'll create the **Deployment**.

This is where things become more interesting.

The Deployment will create and manage **Pods** running our Flask application.

We'll cover:

* What is a Pod?
* What is a Deployment?
* Why don't we create Pods directly?
* Replicas
* Labels
* Selectors
* Rolling updates
* Resource requests/limits
* Environment variables from ConfigMap
* `fieldRef`
* Liveness probe
* Readiness probe
* Pod anti-affinity

Then we'll verify that our Blue Team Pods are actually running.
