
---

#  What Is Kubernetes Really Doing?

Kubernetes is an **orchestration engine**.

Its job is not just to run containers.

Its real job is to:

> Maintain the desired state of your application.


<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/ee73b220-3b4f-471c-810b-2809fd614f42" />


---

##  Step 1 – Containers Run Inside Pods

We already know:

* Applications run inside **containers**
* Containers run inside **Pods**
* Pods run inside a Kubernetes cluster

So if we want our application to run properly, we must tell Kubernetes:

* How many Pods should run?
* Which container image should be used?
* Which port should be exposed?
* How many replicas should exist?
* How should the app be deployed?

These are your **requirements**.

This is your **desired state**.

---

##  Step 2 – Kubernetes Maintains Desired State

Kubernetes continuously monitors the cluster.

If you say:

> “I want 3 login Pods running”

And one Pod crashes…

Kubernetes automatically creates a new one.

Why?

Because its job is to maintain what you declared.

That is orchestration.

---

##  Step 3 – How Do You Tell Kubernetes Your Desire?

Kubernetes cannot read your mind.

You must tell it what you want.

There are **two ways** to do that:

---

# 1️ Imperative Way

You give direct commands:

```bash
kubectl run login-app --image=nginx
kubectl scale deployment login-app --replicas=3
```

Here you are telling Kubernetes step by step what to do.

###  Problem with Imperative:

* No documentation
* Hard to track changes
* Not version-controlled
* Not ideal for production

It works for quick testing.

But not recommended for real systems.

---

# 2️ Declarative Way (Recommended)

You define everything inside a YAML file.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: login-app
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: login
          image: nginx
```

Then apply it:

```bash
kubectl apply -f login.yaml
```

Now Kubernetes reads this file and understands:

* You want 3 replicas
* You want this image
* You want this configuration

And it will continuously try to maintain it.

---

#  Step 4 – Kubernetes Objects

When you give YAML to Kubernetes, it creates an **Object**.

An Object is simply:

> A record of your desired state stored inside Kubernetes.

Examples of objects:

* Pod
* Deployment
* ReplicaSet
* Service
* StatefulSet
* DaemonSet

Each object represents something Kubernetes must manage.

---

#  Step 5 – Reconciliation Loop (The Core Engine)

Inside Kubernetes, controllers are constantly doing this:

```
Desired State  → Compare → Actual State
```

If they are different, Kubernetes fixes it.

This is called the **Reconciliation Loop**.

Example:

You said: 3 replicas
Actual running: 2

Kubernetes creates 1 more Pod automatically.

---

#  Big Picture Example (Pizza System)

Suppose you have:

* Login service
* Payment service
* Delivery service

You declare:

* Login → 3 replicas
* Payment → 2 replicas
* Delivery → 1 replica
* Port 80 exposed
* Image version 1.2.3

Kubernetes stores this as objects.

Then it continuously ensures:

> “Whatever you declared must remain true.”

If a Pod dies → it recreates it.
If a node fails → it reschedules.
If something changes → it corrects it.

That is orchestration.

---

#  Final Clean Definition

Kubernetes is an orchestration engine that:

1. Accepts your desired state (via objects)
2. Stores it
3. Continuously compares actual state with desired state
4. Takes corrective action to maintain that state

---

> Kubernetes does not care how your application runs.
> It only cares that the declared state must always be true.

---


