###                                                                           **<u>Kubernetes Storage</u>**



​                                                                                                                                      <img src="https://cdn-images-1.medium.com/max/1200/1*wZ-fEBj6-j-SZp51b3WyjQ.png" alt="img" style="zoom:67%;" />





In real world, any applications perform either read or write operations, over the data. So, we need some mechanism to store data for the application running inside the Kubernetes cluster.  There are two ways you can storage data in Kubernetes:

**Note: For any kind of volume in a given pod, data is preserved across container restarts.**

So, let's first understand what is Volume?

We've seen in Docker, if you stores data locally, it will be lost if the containers crashes. The new container that would replace this container will have no previous data, so it would be a complete loss of the data. Thus, we cannot rely on containers  for storage of data.

Also, in case of the POD, we can have multiple containers running inside the same POD, so, there will be where a sidecar  container access/process the data produced by the main application container.

To solve above 2 mentioned problems, Kubernetes **Volumes** comes for the rescue. Kubernetes Volume is exposed to the applications as an abstraction, which eventually stores the data on the physical storage that you have provided. At its core,a volume could be a directory or a LUN or a Block Volume or a NFS file share etc etc...

  

​                                                      <img src="https://cdn-images-1.medium.com/max/1200/1*6IsF5gI634ZOI6fg9pCkUg.png" alt="img" style="zoom:50%;" />                                                      

**Types of Volumes**:

1: **Ephemeral Volumes**: This volume is dependent the on the lifecycle of the POD independent of container's lifecycle  running inside it.  You can also share the data between the containers inside the same POD. The Only issue with this type of the volume, you will loose your data if POD crashes.  So, not a good use case for any production kind of workload.

2: **Persistent Volumes** (PV): Independent of the pod's life cycle (depending upon the RECLAIM POLICY), so you will have your data available even the POD got deleted. There are different types of PV 



**Ephemeral Volumes**

**emptyDir** 

An emptyDir an empty volume that got created when a Pod is assigned to a node. It exists as long as that Pod is running on that node. All containers in the Pod can read and write the same files in the emptyDir volume, even though the mount point is different in each container. When a Pod is removed from a node for any reason, the data in the emptyDir is deleted permanently.

**hostPath**

A hostPath volume mounts a file or directory from the host node's filesystem into your Pod. For example in case of Docker , it uses a hostPath of /var/lib/docker. HostPath volumes present many security risks, and it is a best practice to avoid the use of HostPaths when possible.



**Persistent Volumes**

So far what we have learn about the volumes (Ephemeral Volumes), the volume/data is available till the life of the POD. And in Kubernetes world, PODs are ephemerals, they come and go. So, this ephemerals volumes are ONLY recommended for the TEST application.

Now the question arises how to keep data persist in Kubernetes, the answer is PersistentVolume, PersistentVolumeClaims and StorageClass. we will go thru all of these one by one. Before that I would like to mention that in Kubernetes you can provision the PersistentVolumes either **Manually** or **Dynamically**.  Also, we will be discussing the  concept of Container Storage Interface (CSI), which is kind of plugin that we have install based on our storage selection.

Manual Provisioning:

<img src="https://cdn-images-1.medium.com/max/1200/1*HaXNVogwnKGmPFW06qglpQ.png" alt="img" style="zoom:67%;" />

**Persistent Volumes**:
A persistent volume represents a piece of storage that has been provisioned for use with Kubernetes pods. A persistent volume can be used by one or many pods, and can be dynamically or statically provisioned.

**PersistentVolumeClaims**:

A PersistentVolumeClaim (PVC) is a request for storage by a user. It is similar to a Pod. Pods consume node resources and PVCs consume PV resources. Pods can request specific levels of resources (CPU and Memory). Claims can request specific  size and access modes (e.g., they can be mounted ReadWriteOnce, ReadOnlyMany or ReadWriteMany.



>See the Storage_Lab.txt
>
>



**Container Storage Interface (CSI)**:
Kubernetes just provides an interface for Pods to consume it. The storage properties like, speed, replication, resiliency etc. all are the storage provides responsibility . For any storage provider to integrate there storage with Kubernetes, they have to write the CSI plugins. 



<img src="https://cdn-images-1.medium.com/max/1200/1*eqd57VleLyYUsDYPUf2dJw.png" alt="img" style="zoom:50%;" />

CSI code maintenance, updates or any kind of the bug fixes is the responsibility of the storage provider, Kubernetes has nothing to do with that. 

https://kubernetes-csi.github.io/docs/

**Dynamic Provisioning:**

Storage Class:

StorageClass is an API object. It provides a way for administrators to describe the "classes" of storage they offer.  Keeping microservices architecture in mind, Storage needs to be dynamic, this is what StorageClass offers out of the box. At large enterprise level application it is not possible for an administrator to provision volumes manually. I tis not at all scalable solution. **Storage class enable dynamic provisioning of the volumes.**

Every cloud provider supports different provisioners, check your cloud documentation for more information.





<img src="https://cdn-images-1.medium.com/max/1200/1*Zg-xAoC__HQnfDn2vt2Zfw.png" alt="img" style="zoom:67%;" />

> See Storage_Lab.txt



So, we have seem that how we can use the different types of Volumes to share temporary data among containers running in the same pod. Also, we have seen that how to persist dat when the pod dies. We have seen the different ways to provision the volume.

