In Kubernetes, **API objects** are the building blocks you use to describe your cluster’s desired state. These objects are stored in etcd and managed through the Kubernetes API.

---

###  Common Kubernetes API Objects

| Category              | API Object                      | Description                                   |
| --------------------- | ------------------------------- | --------------------------------------------- |
| **Workloads**         | Pod                             | Smallest deployable unit                      |
|                       | ReplicaSet                      | Ensures a specified number of pod replicas    |
|                       | Deployment                      | Manages ReplicaSets with rolling updates      |
|                       | StatefulSet                     | Like Deployment, but for ordered, unique pods |
|                       | DaemonSet                       | Runs a pod on each node                       |
|                       | Job                             | One-time task                                 |
|                       | CronJob                         | Repeating scheduled job                       |
|                       | ReplicationController           | Legacy version of ReplicaSet                  |
| **Service Discovery** | Service                         | Exposes a set of pods as a network service    |
|                       | Endpoint, EndpointSlice         | Holds info on pod IPs for Services            |
|                       | Ingress                         | Manages external access via HTTP(S)           |
| **Config & Secrets**  | ConfigMap                       | Stores non-sensitive configuration            |
|                       | Secret                          | Stores sensitive data like passwords          |
| **Storage**           | PersistentVolume (PV)           | Represents a piece of storage                 |
|                       | PersistentVolumeClaim (PVC)     | Requests storage                              |
|                       | StorageClass                    | Defines how storage is provisioned            |
| **RBAC & Auth**       | Role, ClusterRole               | Defines access rules                          |
|                       | RoleBinding, ClusterRoleBinding | Binds roles to users or groups                |
| **Namespace**         | Namespace                       | Virtual cluster within a cluster              |
| **Networking**        | NetworkPolicy                   | Controls pod-to-pod communication             |
| **Custom**            | CustomResourceDefinition (CRD)  | Add your own API objects                      |

---

###  How to View API Objects in Your Cluster

#### 1. List All API Resources

```bash
kubectl api-resources
```

This shows **names, short names, kinds, and whether namespaced**.

Example:

```
NAME                SHORTNAMES   APIVERSION         NAMESPACED   KIND
pods                po           v1                 true         Pod
deployments         deploy       apps/v1            true         Deployment
```

---

#### 2. List All Available API Versions

```bash
kubectl api-versions
```

Example:

```
apps/v1
batch/v1
v1
rbac.authorization.k8s.io/v1
```

---

#### 3. Get Schema and Fields of Any Object

```bash
kubectl explain <object>
```

Example:

```bash
kubectl explain deployment
```

To go deeper:

```bash
kubectl explain deployment.spec.template.spec.containers
```

---

###  Bonus: Get Full OpenAPI Spec

To see the full OpenAPI schema of all K8s objects:

```bash
kubectl get --raw /openapi/v2 | jq .
```

---


