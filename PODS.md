# PODS



- PODS are the atom of Kubernetes cluster. 
- In Kubernetes instead of deploying the containers individually, we deploy Pods. 
- *Pods* are the smallest deployable units of computing that you can create and manage in Kubernetes.
- A pod can have any number of containers running in it. 
- A pod is basically a wrapper around containers running on a node. 



Why POD?  
The answer is quite simple. For an application user, the container is kind of Virtual Machine, the end user can login, install some packages, stop/start it, so it behaves like an VM to the end user. Also, the containers are designed to run a single process per container. What is an Application requires multiple processes that communicate via IPC or through local files, so in that case they need to run on the same machine(or VM). 
As mentioned grouping multiple processes in a single container is NOT the best practice, so what the solution is ?? This is where PODs comes into the picture.  So, POD 



![img](https://cdn-images-1.medium.com/max/800/1*dz2zMoRYPF7969OY4J2FBg.png)

- POD can have one or more containers running inside it (It doesn't mean that you have to always run multiple containers inside the POD). POD allows you to run closely related process together and provide them same environment. 
- Containers in a pod have shared volumes, Linux namespaces, and cgroups. Each pod has a unique IP address and the port space is shared by all the containers in that pod. This means that different containers inside a pod can communicate with each other using their corresponding ports on localhost

![img](https://cdn-images-1.medium.com/max/800/0*Kb2wsn3ZfmSyG_iY.jpg)

- POD always run on a single worker node,it never spans across multiple worker nodes.
- Containers running inside the POD share the same Network Namespace (Linux) i.e. they share the same IP address, same loopback NIC and PORT space.
- All PODs in a K8s cluster (irrespective of the worker nodes ), resides in a single flat shared Network space, i.e. every Pod can access every Pod with its IP address, no NATING is require, it is just like they are connected over same LAN.

![img](https://cdn-images-1.medium.com/max/1200/1*6U9j1CeOvjNjit6eUEZgNA.png)



```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-name
spec:
  containers:
  - name: container1
    image: image1
  - name: container2
    image: image2
```



**apiVersion**: Version of the Kubernetes API we are going to use.

**kind**: The kind of Kubernetes object we are trying to create, which is a Pod in this case.

**metadata**: Metadata or information that uniquely identifies the object we're creating.

**spec**: Specification of our pod, such as container name, image name, volumes, and resource requests.

Note: **apiVersion**, **kind**, and **metadata** apply to all types of Kubernetes objects and are required fields.  **spec** is also a required field; however, its layout is different for different types of objects.



```yaml
Eg 1: A simple Pod having Single Container:
---------------------------------------------

$ vi container-pod1.yaml 

apiVersion: v1
kind: Pod
metadata:
  name: mypod1
spec:
  containers:
  - name: mycontainer
    image: nginx


kubectl create -f container-pod1.yaml
kubectl get pods
kubectl describe pod mypod1

```

```yaml
Creating a Pod in a Different Namespace by Specifying the Namespace in the Pod Configuration YAML file:
----------
kubectl get namespaces

vi container-pod12.yaml

apiVersion: v1
kind: Pod
metadata:
  name: pod2
  namespace: kube-public
spec:
  containers:
  - name: container2
    image: nginx


kubectl create -f container-pod12.yaml
kubectl --namespace kube-public get pods

```

```yaml
Pod Running a Container that Exposes a Port:
----------------------------------------------------------
vi container-pod3.yaml


apiVersion: v1
kind: Pod
metadata:
  name: pod3
spec:
  containers:
    - name: container3
      image: nginx
      ports:
        - containerPort: 80

kubectl create -f container-pod3.yaml
kubectl logs port3
```

```yaml

Pod with Multiple Containers :
=========================================================
vi container-pod4.yaml

apiVersion: v1

kind: Pod

metadata:

  name: pod4

spec:

  containers:

    - name: first-container

      image: nginx

    - name: second-container

      image: ubuntu

      command:

        - /bin/bash

        - -ec

        - while :; do echo '.'; sleep 5; done



kubectl create -f container-pod4.yaml
kubectl describe pod pod4
kubectl logs <pod-name> <container-name>
kubectl logs pod4 second-container -f
```



```
Pod Running a Container with Resource Requirements

vi container-pod4.yaml

apiVersion: v1
kind: Pod
metadata:
  name: pod4
spec:
  containers:
    - name: container4
      image: nginx
      resources:
        limits:
          memory: "128M"
          cpu: "1"
        requests:
          memory: "64M"
          cpu: "0.5"
```



**Life Cycle of a Pod**

- **Pending**: This means that the pod has been submitted to the cluster, but the controller hasn't created all its containers yet.
- **Running**: This state means that the pod has been assigned to one of the cluster nodes and at least one of the containers is either running or is in the process of starting up.
- **Succeeded**: This state means that the pod has run, and all of its containers have been terminated with success.
- **Failed**: This state means the pod has run and at least one of the containers has terminated with a non-zero exit code,
- **Unknown**: This means that the state of the pod could not be found. This may be because of the inability of the controller to connect with the node that the pod was assigned to.



```shell
More COmmands

$ kubectl get pods

$ kubectl explain pods

$ kubectl explain pod.spec

$ kubectl create -f mypod.yaml

$ kubectl get po mypod -o yaml
$ kubectl get po mypod -o json


Retrieving a pod’s log with kubectl logs:

To see your pod’s log ot the container’s log you run:
$ docker logs <container id>
$ kubectl logs mypod

```

