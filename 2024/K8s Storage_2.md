How to  create a PV YAML using a local directory as storage, followed by a PVC to claim that PV, and finally a Deployment that mounts this PV to a NGINX container.

### PV YAML using Local Directory as Storage:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  hostPath:
    path: /mnt/data
```

### PVC to Claim the PV:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: local-storage
```

### Deployment with NGINX Container Mounting the PV:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      volumes:
        - name: my-pv-storage
          persistentVolumeClaim:
            claimName: my-pvc
      containers:
        - name: nginx
          image: nginx
          volumeMounts:
            - mountPath: "/usr/share/nginx/html"
              name: my-pv-storage
```

In this example:

- The PV YAML defines a PersistentVolume named "local-pv" with 5Gi storage capacity, using a local directory `/mnt/data` as storage.
- The PVC YAML creates a PersistentVolumeClaim named "my-pvc" requesting 2Gi of storage and using the "local-storage" StorageClass.
- The Deployment YAML creates a NGINX Deployment with one replica, which mounts the PVC claimed by "my-pvc" to the NGINX container's `/usr/share/nginx/html` directory.
