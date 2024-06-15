Below are the steps to create an Nginx deployment and a NodePort service with the port specified in the YAML files.

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

### Step 3: Create the NodePort Service YAML

Create a YAML file named `nginx-service.yaml` for the NodePort service. This service will expose the Nginx deployment externally on a specified port.

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
      nodePort: 30007  # specify the node port here
  type: NodePort
```

This service will expose port 80 of the Nginx pods on port 30007 on each node in the cluster.

### Step 4: Apply the Deployment and Service YAML files

Use `kubectl` to apply the YAML files you created.

1. Apply the Nginx deployment:

   ```sh
   kubectl apply -f nginx-deployment.yaml
   ```

2. Apply the NodePort service:

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

   You should see the `nginx-service` listed with a NodePort.

### Step 6: Test the Service

To test the service, you can access the Nginx service externally using the NodePort on any node's IP address.

1. Get the IP address of one of your nodes:

   ```sh
   kubectl get nodes -o wide
   ```

2. Use `curl` or a web browser to access the Nginx service:

   ```sh
   curl http://<node-ip>:30007
   ```

   Replace `<node-ip>` with the IP address of any of your cluster nodes.

You should see the default Nginx welcome page.

### Summary

You have successfully created an Nginx deployment and a NodePort service in Kubernetes. This setup allows you to deploy Nginx pods and expose them externally using a specified NodePort.
