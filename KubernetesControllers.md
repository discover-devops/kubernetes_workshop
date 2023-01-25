![image-20220111131912205](C:\Users\rahchaub\AppData\Roaming\Typora\typora-user-images\image-20220111131912205.png)

For any production type workload, just to meet the business objective  you would always consider to have more than one replica of your PODS. This you full fill the HA requirement as well as performances by offering load balancing for your application. In Kubernetes this can be achieved by with the help of controllers.

What is controller?
A controller is an object that ensures that your application runs in the desired state for its entire runtime.

Kubernetes supports different controllers that you can use for replication, such as ReplicaSets, Deployments, DaemonSets, StatefulSets, and Jobs. 

**ReplicaSets**

For any kind of work load what generally we require:
1: We want to have at least two replicas of the application in case one of them fails or dies unexpectedly. 
2: We would also want the failed replica to recover on its own. 
3: If in case the incoming traffic starts growing, we would want to increase the number of Pods (replicas) running our application.

This is what Replicasets offers in Kubernetes, "**A ReplicaSet is a Kubernetes controller that keeps a certain number of Pods running at any given time**". ReplicaSet acts as a supervisor for multiple Pods across the different nodes in a Kubernetes cluster. A ReplicaSet will terminate or start new Pods to match the configuration specified   the ReplicaSet template.

**Deployment**
A Deployment is a Kubernetes object that acts as a wrapper around a ReplicaSet. it maintains a history of revisions. Every time a change is made to the ReplicaSet or the underlying Pods, a new revision of the ReplicaSet is recorded by the Deployment.  This way, using a Deployment makes it easy to roll back to a previous state or version.

**Strategy**: which strategy the Deployment should use when it replaces old pods with new ones.
**RollingUpdate**: This is a strategy used to update a Deployment without having any downtime. With the RollingUpdate strategy, the controller updates the Pods one by one.
Hence, at any given time, there will always be some Pods running. This strategy is particularly helpful when you want to update the Pod template without incurring any downtime for your application. With rolling update means that there may be two different versions of Pods (old and new) running at the same time.

**maxUnavailable** is the maximum number of Pods that can be unavailable during the update.
**maxSurge** is the maximum number of Pods that can be scheduled/created above the desired number of Pods (as specified in the replicas field).

**Recreate**: In this strategy, all the existing pods are killed before creating the new Pods with an updated configuration. So there will be some downtime, so at any point of time, all the pods will be running with the updated version either OLD or New.
**Use case**: Application Pods that need to have a shared state and so we can't have two different versions of Pods running at the same time.

Rolling Back a Deployment
In real world scenario, we do make code modifications and upgrade of the existing application. It could be patch update of any kind of the bug fix. So time we have to even roll back to the previous version, this is what we can also achieve with the Deployment. We can use the kubectl rollout command to check the revision history and rollback.

**StatefulSets**: It is a Kubernetes objects that is used to deploy stateful application in kubernetes.  
Stateful application : Any application that keep stores data to keep track of its state is a Stateful application, best example would be Database like MySQL, Oracle, MySQL etc.
Stateless application: They don't keep track of the state of the application. Each request is completely new and it is handle by the informatin comes along with the application.

![image-20220111135044117](C:\Users\rahchaub\AppData\Roaming\Typora\typora-user-images\image-20220111135044117.png)



Consider a Java app  talking to Oracle DB , JAVA App depends upon the payload of the requests, these requests could be updating the  data in DB or query the DB. So when Java app connect to DB, it update/query the data based on the previous state of the DB. 

So for the stateless application, we deploy deployment and replicate it based on the requirement. Similarly the staeful application are deployed using StatefulSets, it replicate the stateful application just like deployment. So both and deployment and StatefulSets managed pods based on the identical container specifications. You can also configure storage with both of them in similar way i.e. you can configure storage with both in similar way.

The challenge with the stateful application is that there replication and deletion operation is not so straight forward like stateless pallication. We have to mainatain some , they can't be created/deleted at the same time. They can't be addressed  randomly. The reason is that, replica pods are not identical, istead they are build with common pod blueprint, the statefulSets maintain a unique identity for each of their Pods. So, even if all the Pods are of identical specs, they are not interchangeable. Each of the Pods has a sticky identity that can be used by the application code to manage the state of the application on a particular Pod. The identity of the PODs are maintained  across it's rescheduling, i.e. if  a POD dies it will be replace with the new POD by keeping its pod-identity.

**Why these pods need this identity** ?
For this we need to understand the Database Scaling first.

**Master -slave architecture** 
Each pod has it's own replica of the storage. In order to maintain the same data, they have to continuously syncronize there data. As master is only allowed to change the data and slaves must knows about each change to be up-to-date. And there is mechanism that mantains these thing. When a new pod replica join the existing set-up, it will creates it's own storage, it will clones the data from the privious pod, not with ANY other PODS. Once it has up-to-date data cloned, then it starts synchronizes with the master and start listing for any changes by the master POD.

The data set shoul be persistent storage (PV) of the StatefulSets, just in case of any system level crash to avoid the data loss. Since each pod has it's own data storage, it's own persistent volumes, this PV has the sysncronize data (i.e. the replicated up-to-date data). This PV also incluseds the state of the POD State (weather it's a MASTER pod or SLAVE, POD identifier and other indivildual characteristics). All this information in the POD's own storage volume.

When POD dies and it re-created the persistent pod identifier make sure that the same storage volume get re-attached with the pod. Since the storage has the state of the POD in addition to the replicated data. And for this re-attachement to work, it is necessary to have remote storage. SO that storage can move from one node to another node. POD Identifier : StatefulSets pod get fixed ordered name. 

**DaemonSets**:

DaemonSets are used to manage the creation of a particular Pod on all or a selected set of nodes in a cluster.
If we configure a DaemonSet to create Pods on all nodes, then if new nodes are added to the cluster, new pods will be created to run on these new nodes.
Use case: Logging, Local data caching, Monitoring

**Jobs**: 
A Job is a supervisor in Kubernetes that can be used to manage Pods that are supposed to run a determined task and then terminate gracefully.
A Job creates the specified number of Pods and ensures that they successfully complete their workloads or tasks.

Use case: Backup operation, 

