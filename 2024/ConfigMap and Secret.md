ConfigMap and Secret are Kubernetes resources used to manage sensitive information and configuration data separately from application code. 

- **ConfigMap**: A ConfigMap is an API object that allows you to store non-sensitive data in key-value pairs. It's commonly used to store configuration files, environment variables, or any other kind of configuration data that your application needs. ConfigMaps decouple configuration data from container images, allowing for more flexible and portable deployments.

- **Secret**: A Secret is similar to a ConfigMap but is specifically designed to store sensitive information such as passwords, API keys, and tokens. Secrets are base64 encoded and stored in etcd, Kubernetes' key-value store, and mounted into pods as files or environment variables.

Here's an example YAML for a ConfigMap and a Secret:

### ConfigMap YAML Example:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-configmap
data:
  server.properties: |
    server.port=8080
    server.host=localhost
    server.debug=false
```

### Secret YAML Example:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: YWRtaW4=  # base64 encoded value of 'admin'
  password: cGFzc3dvcmQ=  # base64 encoded value of 'password'
```

In the ConfigMap example:
- We define a ConfigMap named "my-configmap" with a key-value pair "server.properties" containing some sample configuration data.

In the Secret example:
- We define a Secret named "my-secret" of type "Opaque" (which means it can contain arbitrary data).
- Inside the Secret, we have base64-encoded values for a username and a password.

Now, let's create a Deployment that uses both the ConfigMap and the Secret:

### Deployment YAML Using ConfigMap and Secret:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-container
          image: my-image
          env:
            - name: SERVER_PORT
              valueFrom:
                configMapKeyRef:
                  name: my-configmap
                  key: server.properties
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: my-secret
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: my-secret
                  key: password
```

In this Deployment YAML:
- We reference the ConfigMap "my-configmap" to set environment variables for the container.
- We reference the Secret "my-secret" to set environment variables for sensitive information like username and password. These are retrieved from the Secret's data field using the specified keys.
