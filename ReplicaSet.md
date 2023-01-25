# ReplicaSet

- In the Pod's section you’ve learned that, pods represent the basic deployable unit in Kubernetes. 
- We have seen  how to create, supervise, and manage them manually. 
- We've also seen the challenges while launching application in pods and managing applications with ONLY pods. 
- So, in the real-world use cases, you want your deployments to stay up and running automatically and remain healthy without any manual intervention. 
- To do this, you almost never create pods directly. Instead, you create other types of resources, such as **ReplicationControllers** or **Deployments**, which then create and manage the actual pods.



##### What is **ReplicationControllers**

- A ReplicationController is a Kubernetes resource that ensures its pods are always kept running. 
- If the pod disappears for any reason, such as in the event of a node disappearing from the cluster or because the pod was evicted from the node, the ReplicationController notices the missing pod and creates a replacement pod.



```yaml

Creating a Simple ReplicaSet with nginx Containers

vi replicaset-nginx.yaml

apiVersion: apps/v1

kind: ReplicaSet

metadata:

  name: nginx-replicaset

  labels:

    app: nginx

spec:

  replicas: 2

  selector:

    matchLabels:

      environment: production

  template:

    metadata:

      labels:

        environment: production

    spec:

      containers:

      - name: nginx-container

        image: nginx
        
        
 kubectl create -f replicaset-nginx.yaml
 kubectl get rs nginx-replicaset
 kubectl get pods
 kubectl describe rs nginx-replicaset
```

```yaml
Deleting Pods Managed by a ReplicaSet
======================================

kubectl get pods
kubectl delete pod <pod_name>
kubectl describe rs nginx-replicaset
kubectl delete rs nginx-replicaset
kubectl get pods
```



```
Scaling a ReplicaSet 
===========================================

vi replicaset-nginx.yaml

apiVersion: apps/v1

kind: ReplicaSet

metadata:

  name: nginx-replicaset

  labels:

    app: nginx

spec:

  replicas: 2

  selector:

    matchLabels:

      environment: production

  template:

    metadata:

      labels:

        environment: production

    spec:

      containers:

      - name: nginx-container

        image: nginx
        
kubectl apply -f replicaset-nginx.yaml
kubectl get pods
kubectl scale --replicas=4 rs nginx-replicaset
kubectl get pods
kubectl scale --replicas=1 rs nginx-replicaset
kubectl get pods
kubectl delete rs nginx-replicaset
```

