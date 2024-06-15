To create an Nginx deployment and a Cluster IP service in Kubernetes, follow the steps outlined below. 

### Step 1: Set up your Kubernetes environment

Ensure you have a running Kubernetes cluster and `kubectl` configured to interact with your cluster.

### Step 2: Create the Nginx Deployment YAML

Create a YAML file named `nginx-deployment.yaml` for the Nginx deployment. This file defines the deployment of Nginx pods.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

This deployment will create 3 replicas of the Nginx pod.

### Step 3: Create the Cluster IP Service YAML

Create a YAML file named `nginx-service.yaml` for the Cluster IP service. This service will expose the Nginx deployment internally within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

This service will route traffic on port 80 to the Nginx pods.

### Step 4: Apply the Deployment and Service YAML files

Use `kubectl` to apply the YAML files you created.

1. Apply the Nginx deployment:

   ```sh
   kubectl apply -f nginx-deployment.yaml
   ```

2. Apply the Cluster IP service:

   ```sh
   kubectl apply -f nginx-service.yaml
   ```

### Step 5: Verify the Deployment and Service

1. Check the status of the deployment:

   ```sh
   kubectl get deployments
   ```

   You should see the `nginx-deployment` listed with the desired number of replicas.

2. Check the status of the pods:

   ```sh
   kubectl get pods
   ```

   You should see 3 running pods for the Nginx deployment.

3. Check the status of the service:

   ```sh
   kubectl get services
   ```

   You should see the `nginx-service` listed with a Cluster IP.

### Step 6: Test the Service

To test the service internally, you can use a temporary pod with `curl` to access the Nginx service.

1. Create a temporary pod:

   ```sh
   kubectl run curlpod --image=radial/busyboxplus:curl -i --tty
   ```

2. Inside the pod, use `curl` to access the Nginx service:

   ```sh
   curl nginx-service
   ```

You should see the default Nginx welcome page HTML.

### Summary

You have successfully created an Nginx deployment and a Cluster IP service in Kubernetes. This setup allows you to deploy Nginx pods and expose them internally within your cluster using a Cluster IP service.
