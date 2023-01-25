​                                                                                                                                    StatefulSets

**Stateful** application : Any application that keep stores data to keep track of its state is a Stateful application, best example would be Database like MySQL, Oracle, MySQL etc.
**Stateless** application: They don't keep track of the state of the application. Each request is completely new and it  is handle by the information comes along with the application.

​                                                    



<img src="https://cdn-images-1.medium.com/max/1200/1*mqq2_qxUalnHBlfAGYH6Jg.png" alt="img" style="zoom:67%;" />



##### How we access server (webserver)

1: User will pass its credential (username and password) in HTTP request session.

2: The Webserver will check with DB and verify the credentials.

3: Once the user is verified, the server will create the session data and stores in the server, like **userA:SessionID.** Server will return some identifier to the user.

4: User is now able to login and server will return some page (home page).

5: So the session for the user got created. User will perform other tasks, such as read the messages etc etc..

This is not the detail explanation, but kind of high level view how the session get created and server stores the session data on the 



#### How to scale the AppServer

1: We have to launch the new App Server and put a LB infront of it.
2: However the newly launched server won't have the session data, it won't able to server the User's request.
3: The newly launched server again, have to perform the above steps.

This is something not a recommended approach. So here comes the concept of Stateless application.







#### HA Solution for DB

\- In order to maintain the HA for the DB we maintain the multiple copy of the DB by running multiple DB servers.

\- We first set-up the Master server and multiple slave server. This is known as single master and multiple slave topology. There could be many other ways too to create the HA, but I just want to keep the things simple.

\- Now READ operations can be perform any of the server, however all the WRITE operations are performed over the MASTER ONLY.







# HA Solution for DB

\- In order to maintain the HA for the DB we maintain the multiple copy of the DB by running multiple DB servers.

\- We first set-up the Master server and multiple slave server. This is known as single master and multiple slave topology. There could be many other ways too to create the HA, but I just want to keep the things simple.

\- Now READ operations can be perform any of the server, however all the WRITE operations are performed over the MASTER ONLY.



<img src="https://cdn-images-1.medium.com/max/1200/1*pQVZmhlS1K8ljvYmoeyVhg.png" alt="img" style="zoom:67%;" />

# Order of the Deployment

\- Master server must be deployed first, then we can deploy the slaves.

\- When the first Slave get deployed , it will perform the initial clone of the DB from the master.

\- Once the Slave is deployed the slave will keep it self in continuous sync with the master server.

\- Now, we will deploy the second slave, this time it will clone it self from the the already existing Slave server, not from the master, so that keep master healthy.

\- When slave 2 is ready, it will keep itself in continuous sync with the master.

\- Similarly, all the other slave servers will get configured.

\- All the Slave servers, configure the master address. 



#### Can't we get this Deployment ?

\- It deploy PODs.

\- With deployment you can't guarantee the order, almost all the PODs came up at the same time.

\- In order to maintain the replication and copy the initial database, we must have to differentiate the pods, Master, Slave-1, Slave-2 and so on, this is not possible with the deployment.

\- We can't provide MASTER a constant hostname or address that should not change.

\- IP address of the POD dynamically assigned and it change when the POD got recreated. Also POD come ups with random name.

\- So, for stseful application Deployment is not a good fit.



# StatefulSets 

\- With deployment you can't guarantee the order, almost all the PODs came up at the same time.

\- In order to maintain the replication and copy the initial database, we must have to differentiate the pods, Master, Slave-1, Slave-2 and so on, this is not possible with the deployment.

\- We can't provide MASTER a constant hostname or address that should not change.

\- IP address of the POD dynamically assigned and it change when the POD got recreated. Also POD come ups with random name.

\- So, for stseful application Deployment is not a good fit.



So for the stateless application, we deploy deployment and replicate it based on the requirement. Similarly the staeful application are deployed using StatefulSets, it replicate the stateful application just like deployment.

So both and deployment and StatefulSets managed pods based on the identical container specifications. You can also configure storage with both of them in similar way i.e. you can configure storage with both in similar way.

The challenge with the stateful application is that there replication and deletion operation is not so straight forward like stateless application. We have to maintain some order, they can't be created/deleted at the same time. They can't be addressed  randomly.

The reason is that, replica pods are not identical, instead they are build with common pod blueprint, the statefulSets maintain a unique identity for each of their Pods.
So, even if all the Pods are of identical specs, they are not interchangeable.

Each of the Pods has a sticky identity that can be used by the application code to manage the state of the application on a particular Pod. The identity of the PODs are maintained across it's rescheduling, i.e. if  a POD dies it will be replace with the new POD by keeping its pod-identity.

**Why these pods need this identity ?**

For this we need to understand the Database Scaling first.

Master -slave architecture 
Each pod has it's own replica of the storage. In order to maintain the same data, they have to continuously syncronize there data. As master is only allowed to change the data and slaves must knows about each change to be up-to-date. And there is mechanism that mantains these thing.

When a new pod replica join the existing set-up, it will creates it's own storage, it will clones the data from the privious pod, not with ANY other PODS. Once it has up-to-date data cloned, then it starts synchronizes with the master and start listing for any changes by the master POD.

The data set should be persistent storage (PV) of the StatefulSets, just in case of any system level crash to avoid the data loss.

Since each pod has it's own data storage, it's own persistent volumes, this PV has the sysncronize data (i.e. the replicated up-to-date data). This PV also includes the state of the POD State (weather it's a MASTER pod or SLAVE, POD identifier and other individual characteristics). All this information stored in the POD's own storage volume.

When POD dies and it re-created the persistent pod identifier make sure that the same storage volume get re-attached with the pod. Since the storage has the state of the POD in addition to the replicated data.

And for this re-attachment to work, it is necessary to have remote storage. SO that storage can move from one node to another node.





![Figure 14.1: Modular stateful components in Kubernetes ](https://learning.oreilly.com/api/v2/epubs/urn:orm:book:9781838820756/files/image/B14870_14_01.jpg)



