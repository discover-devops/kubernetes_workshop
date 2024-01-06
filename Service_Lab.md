
### Nginx Deployment

**nginx-deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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

Apply the deployment with:

```bash
kubectl apply -f nginx-deployment.yaml
```

### Nginx ClusterIP Service

**nginx-service-clusterip.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-clusterip
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply the ClusterIP service with:

```bash
kubectl apply -f nginx-service-clusterip.yaml
```

### Nginx NodePort Service

**nginx-service-nodeport.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-nodeport
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
```

Apply the NodePort service with:

```bash
kubectl apply -f nginx-service-nodeport.yaml
```

After applying these configurations, you'll have a Nginx Deployment with three replicas, and two Services: one using ClusterIP and the other using NodePort.

- The ClusterIP Service allows internal communication within the cluster. You can access the Nginx service using its ClusterIP.

- The NodePort Service exposes the Nginx service on a port on each node, making it accessible externally. You can use the NodePort and the IP of any node to access the Nginx service from outside the cluster.


 YAML example for the NodePort Service with the explicit specification of the NodePort:

**nginx-service-nodeport.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-nodeport
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
  nodePort: 30080   # Specify the NodePort value, adjust as needed
```
