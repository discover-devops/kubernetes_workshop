### Step-by-Step Guide to Deploy a Microservice Application on AWS EKS

This guide walks you through deploying a simple microservice application on a Kubernetes cluster hosted on AWS EKS. The application will consist of a frontend service, a backend service, and a database. We will create Deployments, Services, ConfigMaps, PersistentVolumes, and PersistentVolumeClaims.

#### Prerequisites

1. AWS CLI installed and configured.
2. `kubectl` installed and configured to interact with your EKS cluster.
3. `eksctl` installed for easy EKS cluster management.

#### Step 1: Create an EKS Cluster

First, create an EKS cluster using `eksctl`.

```sh
eksctl create cluster --name my-cluster --region us-west-2 --nodegroup-name my-nodes --node-type t3.medium --nodes 3
```

#### Step 2: Create Namespaces

Namespaces help organize resources within the cluster.

```sh
kubectl create namespace dev
kubectl create namespace prod
```

#### Step 3: Create ConfigMaps and Secrets

Create ConfigMaps for application configuration and Secrets for sensitive information like database passwords.

**ConfigMap for Backend Service:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: dev
data:
  DATABASE_HOST: "mysql"
  DATABASE_NAME: "mydb"
```

**Secret for Database Credentials:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: dev
type: Opaque
data:
  username: c3Fsb2cx
  password: cGFzc3dvcmQ=
```

Save these files as `backend-configmap.yaml` and `db-secret.yaml` and apply them:

```sh
kubectl apply -f backend-configmap.yaml
kubectl apply -f db-secret.yaml
```

#### Step 4: Create PersistentVolumes and PersistentVolumeClaims

PersistentVolumes (PVs) provide storage to be used by PersistentVolumeClaims (PVCs).

**PersistentVolume:**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  awsElasticBlockStore:
    volumeID: vol-0c3a6dfexample
    fsType: ext4
```

**PersistentVolumeClaim:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
  namespace: dev
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Save these files as `pv.yaml` and `pvc.yaml` and apply them:

```sh
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
```

#### Step 5: Create Deployments and Services

**Backend Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: my-backend:latest
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: backend-config
        env:
        - name: DATABASE_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

**Backend Service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: dev
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

**Frontend Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: my-frontend:latest
        ports:
        - containerPort: 80
```

**Frontend Service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: dev
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

**Database Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: pvc-data
```

**Database Service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: dev
spec:
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
  type: ClusterIP
```

Save these files as `backend-deployment.yaml`, `backend-service.yaml`, `frontend-deployment.yaml`, `frontend-service.yaml`, `mysql-deployment.yaml`, and `mysql-service.yaml`, and apply them:

```sh
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml
```

#### Step 6: Verify Deployments and Services

Check the status of your Deployments and Services:

```sh
kubectl get deployments -n dev
kubectl get services -n dev
kubectl get pods -n dev
```

#### Step 7: Access the Application

Since the frontend service is of type `LoadBalancer`, it will provision an external IP. You can access the frontend application using this IP:

```sh
kubectl get services -n dev
```

Look for the `EXTERNAL-IP` column for the `frontend` service and open it in your web browser.

### Summary

This guide covers the following steps to deploy a microservice application on AWS EKS:
1. Creating an EKS cluster.
2. Creating namespaces.
3. Creating ConfigMaps and Secrets.
4. Creating PersistentVolumes and PersistentVolumeClaims.
5. Creating Deployments and Services for the backend, frontend, and database components.
6. Verifying the deployments and services.
7. Accessing the application.

By following these steps, you can deploy a scalable and resilient microservice application on a Kubernetes cluster managed by AWS EKS.
