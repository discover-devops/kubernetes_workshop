# Kubernetes Custom Resource Definitions (CRD) -- Hands-on Runbook

## Objective

By the end of this lab, you will understand:

-   What problem CRDs solve
-   What is a Custom Resource Definition (CRD)
-   What is a Custom Resource (CR)
-   Why creating a CR does **not** create Pods
-   How Operators fit into the picture
-   How CRDs compare with built-in Kubernetes resources such as
    Deployments

------------------------------------------------------------------------

# Prerequisites

-   Minikube cluster (or any Kubernetes cluster)
-   `kubectl` configured
-   Basic knowledge of Pods and Deployments

Verify your cluster:

``` bash
kubectl get nodes
kubectl cluster-info
```

------------------------------------------------------------------------

# 1. The Problem

Kubernetes already understands resources like:

-   Pod
-   Deployment
-   Service
-   ConfigMap
-   StatefulSet

But suppose a platform team wants developers to create Redis using a
simple YAML like:

``` yaml
kind: RedisCache
```

If you run:

``` bash
kubectl get rediscaches
```

You will see:

``` text
error: the server doesn't have a resource type "rediscaches"
```

Reason:

Kubernetes has never heard about **RedisCache**.

------------------------------------------------------------------------

# 2. Solution -- Custom Resource Definition (CRD)

A CRD extends the Kubernetes API by introducing a brand-new resource
type.

Create **redis-crd.yaml**

``` yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition

metadata:
  name: rediscaches.cache.hotstar.internal

spec:
  group: cache.hotstar.internal

  scope: Namespaced

  names:
    plural: rediscaches
    singular: rediscache
    kind: RedisCache
    shortNames:
      - rc

  versions:
    - name: v1
      served: true
      storage: true

      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - image
                - replicas

              properties:
                image:
                  type: string
                replicas:
                  type: integer
                tier:
                  type: string
                maxmemory:
                  type: string
```

Apply it:

``` bash
kubectl apply -f redis-crd.yaml
```

Verify:

``` bash
kubectl get crd
kubectl get crd rediscaches.cache.hotstar.internal
kubectl api-resources | grep redis
kubectl get rediscaches
```

Expected:

``` text
No resources found
```

This means Kubernetes now **recognizes** the resource, but no instances
exist yet.

------------------------------------------------------------------------

# 3. Create a Custom Resource (CR)

Create **redis-cache.yaml**

``` yaml
apiVersion: cache.hotstar.internal/v1
kind: RedisCache

metadata:
  name: watchlist-cache

spec:
  image: redis:7
  replicas: 3
  tier: prod
  maxmemory: "256mb"
```

Apply it:

``` bash
kubectl apply -f redis-cache.yaml
```

Verify:

``` bash
kubectl get rediscaches
kubectl describe rediscache watchlist-cache
```

------------------------------------------------------------------------

# 4. Important Observation

Check whether Pods were created.

``` bash
kubectl get pods
kubectl get all
```

Expected:

``` text
No resources found
```

Why?

Because a CRD only teaches Kubernetes a **new resource type**.

It does **not** define what action should happen when a new resource is
created.

------------------------------------------------------------------------

# 5. Where Does the Operator Fit?

The Operator continuously watches Custom Resources.

For example:

    RedisCache Created
            │
            ▼
    Redis Operator
            │
    Creates StatefulSet
            │
    Creates Service
            │
    Creates PVC
            │
    Pods Running

In this lab we did **not** build a real Operator.

Instead, we manually simulated its behavior.

------------------------------------------------------------------------

# 6. Built-in Resources vs Custom Resources

## Deployment Flow

    deployment.yaml
          │
    kubectl apply
          │
    API Server
          │
    etcd
          │
    Deployment Controller
          │
    ReplicaSet
          │
    ReplicaSet Controller
          │
    Pods

## CRD Flow

    redis-cache.yaml
          │
    kubectl apply
          │
    API Server
          │
    etcd
          │
    Redis Operator
          │
    StatefulSet / Deployment
          │
    Pods

------------------------------------------------------------------------

# Comparison

  Built-in Kubernetes                Custom Kubernetes
  ---------------------------------- ----------------------------------
  Deployment                         RedisCache
  Definition built into API Server   Definition added using CRD
  Deployment Controller              Redis Operator
  Creates ReplicaSet                 Creates StatefulSet / Deployment
  Produces Pods                      Produces Pods

------------------------------------------------------------------------

# Key Takeaways

-   CRD extends the Kubernetes API.
-   CR is an instance of that new resource.
-   CRDs do not create Pods.
-   Operators watch Custom Resources and reconcile the desired state.
-   Built-in resources and custom resources follow the same
    architecture---the only difference is that custom resources require
    a custom controller (Operator).

------------------------------------------------------------------------

# Cleanup

``` bash
kubectl delete -f redis-cache.yaml
kubectl delete -f redis-crd.yaml
```

Verify:

``` bash
kubectl get crd | grep redis
kubectl get rediscaches
```
