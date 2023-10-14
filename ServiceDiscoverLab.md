
# Kubernetes Labs

In this repository, you'll find instructions and YAML files for two Kubernetes labs. 

## Lab 1: Creating a NodePort Service with Nginx Containers

**my-nginx-deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nginx
      environment: production
  template:
    metadata:
      labels:
        app: nginx
        environment: production
    spec:
      containers:
      - name: nginx-container
        image: nginx
```

**nginx-service-nodeport.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-nodeport
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32023
  selector:
    app: nginx
    environment: production
```

...

## Lab 2: Creating a ClusterIP Service with Nginx Containers

**nginx-deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nginx
      environment: production
  template:
    metadata:
      labels:
        app: nginx
        environment: production
    spec:
      containers:
      - name: nginx-container
        image: nginx
```

**nginx-service-clusterip.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-clusterip
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: nginx
    environment: production
```

...

```

You can create a Markdown file (e.g., `k8s-labs.md`) and paste the YAML code as shown above to document your Kubernetes labs. This format makes it easy to read and understand the content of your YAML files within the Markdown document.
