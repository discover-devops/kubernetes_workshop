# Lab 2 — Deploy a Sample Application with New Relic APM

## Objective

In Lab 1, we connected our Kubernetes cluster to New Relic.

We can now see Kubernetes infrastructure such as:

* Nodes
* Pods
* Deployments
* Kubernetes metrics
* Kubernetes events
* Container logs

But infrastructure monitoring alone does not tell us everything about an application.

In this lab, we will deploy a small Node.js application and instrument it with the **New Relic APM agent**.

At the end of this lab, we should be able to see the application inside:

**New Relic → APM & Services**

and observe:

* Application transactions
* Request latency
* Application errors
* Distributed tracing information

---

# 1. What are we deploying?

We are deploying a very small Node.js application called:

```text
demo-app
```

The application has three HTTP endpoints.

| Endpoint     | Purpose                          |
| ------------ | -------------------------------- |
| `/api/fast`  | Normal/fast request              |
| `/api/slow`  | Simulates a slow request         |
| `/api/error` | Intentionally generates an error |

The application is intentionally simple because we want to generate different types of telemetry for New Relic.

The architecture is:

```text
User
 |
 v
demo-app Service
 |
 +----------------------+
 |                      |
 v                      v
Pod 1                  Pod 2
 |                      |
 +----------+-----------+
            |
       Node.js App
            |
       New Relic APM
            |
            | telemetry
            v
        New Relic
```

We deploy **2 replicas** of the application so that Kubernetes runs two application Pods.

---

# 2. What is a Kubernetes Deployment?

Before creating the application, understand what this object does.

A Kubernetes **Deployment** manages application Pods for us.

For example:

```yaml
spec:
  replicas: 2
```

means:

> "Kubernetes should keep two Pods running for this application."

So we don't manually create two Pods.

We tell Kubernetes:

```text
I want 2 copies of demo-app.
```

Kubernetes creates and maintains them.

If one Pod dies:

```text
Pod 1   Running
Pod 2   Running

       ↓ Pod 1 crashes

Pod 1   Failed
Pod 2   Running

       ↓

Kubernetes creates another Pod
```

The Deployment gives us the desired state.

---

# 3. What is the New Relic APM agent?

The Kubernetes integration from Lab 1 monitors the **Kubernetes environment**.

For example:

```text
Node CPU
Node Memory
Pod CPU
Pod Memory
Pod restarts
Kubernetes Events
```

The APM agent monitors the **application**.

Conceptually:

```text
Kubernetes Integration
        ↓
   Kubernetes
   visibility


New Relic APM Agent
        ↓
   Application
   visibility
```

In our Node.js application, this line loads the New Relic APM agent:

```javascript
require('newrelic');
```

The agent observes application activity such as HTTP transactions, latency and errors and sends that telemetry to New Relic.

---

# 4. Create the application YAML

We will create one YAML file containing:

1. Namespace
2. ConfigMap
3. Deployment
4. Service

The ConfigMap contains the application code.

The Deployment runs the application.

The Service gives the application a stable Kubernetes network endpoint.

## Create the file

Run the following command exactly:

```bash
cat <<'EOF' > sample-app-deployment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-app
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-code
  namespace: demo-app
data:
  app.js: |
    require('newrelic');
    const express = require('express');
    const app = express();

    app.get('/api/fast', (req, res) => {
      res.json({
        message: 'Fast endpoint',
        timestamp: new Date()
      });
    });

    app.get('/api/slow', async (req, res) => {
      await new Promise(resolve => setTimeout(resolve, 2000));

      res.json({
        message: 'Slow endpoint',
        timestamp: new Date()
      });
    });

    app.get('/api/error', (req, res) => {
      throw new Error('Intentional error for demo');
    });

    app.listen(3000, () => {
      console.log('App running on port 3000');
    });

  package.json: |
    {
      "name": "demo-app",
      "version": "1.0.0",
      "dependencies": {
        "express": "^4.18.0",
        "newrelic": "^11.0.0"
      }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: app
          image: node:18-alpine
          command:
            - /bin/sh
            - -c
          args:
            - |
              mkdir -p /app
              cp /config/app.js /app/app.js
              cp /config/package.json /app/package.json
              cd /app
              npm install
              node app.js
          env:
            - name: NEW_RELIC_LICENSE_KEY
              valueFrom:
                secretKeyRef:
                  name: newrelic-license
                  key: licenseKey
            - name: NEW_RELIC_APP_NAME
              value: demo-app
            - name: NEW_RELIC_DISTRIBUTED_TRACING_ENABLED
              value: "true"
            - name: NEW_RELIC_LOG_LEVEL
              value: info
            - name: NEW_RELIC_LABELS
              value: "environment:demo;team:platform"
          volumeMounts:
            - name: config
              mountPath: /config
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
      volumes:
        - name: config
          configMap:
            name: app-code
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app
  namespace: demo-app
spec:
  selector:
    app: demo-app
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
EOF
```

### Important

The original lab assumes that `/app` already exists inside the Node.js container.

Our `node:18-alpine` image does not guarantee that directory exists.

Therefore we explicitly create it:

```bash
mkdir -p /app
```

This prevents the application from failing during startup.

---

# 5. Verify the YAML file

Run:

```bash
ls -lh sample-app-deployment.yaml
```

You should see the file.

Then validate the Kubernetes configuration without actually creating anything:

```bash
kubectl apply --dry-run=client -f sample-app-deployment.yaml
```

You should see resources such as:

```text
namespace/demo-app configured
configmap/app-code created
deployment.apps/demo-app created
service/demo-app created
```

If there are no errors, continue.

---

# 6. Apply the application

Run:

```bash
kubectl apply -f sample-app-deployment.yaml
```

This creates:

```text
Namespace
   ↓
ConfigMap
   ↓
Deployment
   ↓
Pods
   ↓
Service
```

---

# 7. Create the New Relic license secret

The application needs the New Relic license key so that the APM agent can send telemetry to New Relic.

The secret must exist in the same namespace as the application:

```text
demo-app
```

Set your license key as an environment variable.

Replace `YOUR_LICENSE_KEY` with your actual New Relic license key.

```bash
export NEW_RELIC_LICENSE_KEY='YOUR_LICENSE_KEY'
```

Then create the Kubernetes Secret:

```bash
kubectl create secret generic newrelic-license \
  --from-literal=licenseKey="$NEW_RELIC_LICENSE_KEY" \
  -n demo-app
```

Verify:

```bash
kubectl get secret newrelic-license -n demo-app
```

Expected:

```text
NAME               TYPE     DATA   AGE
newrelic-license   Opaque   1      ...
```

Do not display or share the actual license key.

---

# 8. Restart the Deployment

The Deployment was created before the Secret was created.

Restart it so that the Pods are created with the New Relic license available:

```bash
kubectl rollout restart deployment demo-app -n demo-app
```

Check the Pods:

```bash
kubectl get pods -n demo-app
```

Wait until both Pods are:

```text
1/1   Running
```

You should eventually see something similar to:

```text
NAME                        READY   STATUS    RESTARTS   AGE
demo-app-xxxxxxxxxx-xxxxx   1/1     Running   0          ...
demo-app-xxxxxxxxxx-yyyyy   1/1     Running   0          ...
```

---

# 9. Verify the Deployment

Run:

```bash
kubectl get deployment -n demo-app
```

Expected:

```text
NAME        READY   UP-TO-DATE   AVAILABLE
demo-app    2/2     2            2
```

This confirms that Kubernetes is maintaining the requested two replicas.

---

# 10. Verify the Service

Run:

```bash
kubectl get service -n demo-app
```

Expected:

```text
NAME        TYPE        CLUSTER-IP      PORT(S)
demo-app    ClusterIP   10.x.x.x        80/TCP
```

Our application itself listens on:

```text
3000
```

The Kubernetes Service exposes it internally on:

```text
80
```

Therefore:

```text
Service :80
     ↓
Pod :3000
```

---

# 11. Test the application inside the EC2 instance

Before exposing anything to the Internet, verify that the application itself works.

Run:

```bash
kubectl port-forward \
  --address 127.0.0.1 \
  -n demo-app \
  svc/demo-app 8080:80
```

Keep this terminal running.

Open another terminal on the EC2 instance and test:

```bash
curl http://127.0.0.1:8080/api/fast
```

Expected response:

```json
{
  "message": "Fast endpoint",
  "timestamp": "..."
}
```

Test the slow endpoint:

```bash
time curl http://127.0.0.1:8080/api/slow
```

This endpoint intentionally waits approximately **2 seconds** before returning.

Test the error endpoint:

```bash
curl http://127.0.0.1:8080/api/error
```

This endpoint intentionally throws an application error.

The error is expected.

---

# 12. Access the application from your laptop using EC2 Public IP

For the live class, we may want to access the application from a browser.

Stop the previous port-forward with:

```text
Ctrl + C
```

Now run:

```bash
kubectl port-forward \
  --address 0.0.0.0 \
  -n demo-app \
  svc/demo-app 8080:80
```

This changes the flow from:

```text
127.0.0.1:8080
      ↓
EC2
```

to:

```text
0.0.0.0:8080
      ↓
EC2
      ↓
Kubernetes Service
      ↓
demo-app
```

---

# 13. Allow port 8080 in the EC2 Security Group

Go to:

**AWS Console → EC2 → Your Instance → Security → Security Groups → Inbound rules**

Add:

```text
Type:       Custom TCP
Port:       8080
Source:     YOUR_PUBLIC_IP/32
```

For example:

```text
203.0.113.25/32
```

Using your own IP is safer than:

```text
0.0.0.0/0
```

If you intentionally want the demo application accessible from anywhere during a class, you can use:

```text
0.0.0.0/0
```

but remember that this exposes port 8080 of the EC2 instance to the Internet.

---

# 14. Access the application from your browser

Find the EC2 public IPv4 address.

For example:

```text
54.x.x.x
```

Then open:

```text
http://EC2_PUBLIC_IP:8080/api/fast
```

Example:

```text
http://54.x.x.x:8080/api/fast
```

You should receive the JSON response.

Test:

```text
http://EC2_PUBLIC_IP:8080/api/slow
```

This should take approximately 2 seconds.

Test:

```text
http://EC2_PUBLIC_IP:8080/api/error
```

This generates an application error.

---

# 15. What kind of load are we generating?

Now we want New Relic to receive enough application activity to display useful APM data.

We will repeatedly call the three endpoints.

Run this in a **second EC2 terminal** while port-forwarding is running:

```bash
for i in {1..50}; do
  curl -s http://127.0.0.1:8080/api/fast > /dev/null
  curl -s http://127.0.0.1:8080/api/slow > /dev/null
  curl -s http://127.0.0.1:8080/api/error > /dev/null || true
  sleep 1
done
```

This is **not a performance/load-testing tool** like JMeter or k6.

It is simply a small traffic generator for our demonstration.

Each loop generates:

```text
1 request → /api/fast
1 request → /api/slow
1 request → /api/error
```

Then waits one second.

So approximately:

```text
50 fast requests
50 slow requests
50 error requests
```

The purpose is to create observable APM data.

---

# 16. Understand what New Relic should receive

Our application is producing three different types of behavior.

### `/api/fast`

```text
Request
   ↓
Fast response
   ↓
Normal transaction
```

### `/api/slow`

```text
Request
   ↓
2-second delay
   ↓
Slow transaction
```

### `/api/error`

```text
Request
   ↓
Application exception
   ↓
Error transaction
```

New Relic APM observes these application transactions.

---

# 17. Verify APM in New Relic

Go to:

**New Relic → APM & Services**

Look for:

```text
demo-app
```

Open `demo-app`.

You should eventually see application activity.

Look for transactions corresponding to:

```text
/api/fast
/api/slow
/api/error
```

The exact transaction names displayed by New Relic may include framework-specific naming.

---

# 18. What should we observe?

### Fast endpoint

Should have relatively low latency.

### Slow endpoint

Should show approximately 2 seconds of application response time because our code deliberately waits:

```javascript
await new Promise(resolve => setTimeout(resolve, 2000));
```

### Error endpoint

Should generate application errors.

Therefore our APM data should contain:

```text
demo-app
   |
   +── Normal transactions
   |
   +── Slow transactions
   |
   +── Application errors
```

---

# 19. If `demo-app` does not appear in APM

First check the Pods:

```bash
kubectl get pods -n demo-app
```

Then check the application logs:

```bash
kubectl logs -n demo-app deployment/demo-app
```

Look for New Relic agent startup messages.

You can also inspect one Pod:

```bash
kubectl logs -n demo-app <POD_NAME>
```

If necessary, check the previous container:

```bash
kubectl logs -n demo-app <POD_NAME> --previous
```

Also verify that the New Relic license secret exists:

```bash
kubectl get secret newrelic-license -n demo-app
```

And verify the application environment:

```bash
kubectl exec -n demo-app deployment/demo-app -- env | grep NEW_RELIC
```

The important things to confirm are:

```text
NEW_RELIC_LICENSE_KEY
NEW_RELIC_APP_NAME=demo-app
```

---

# 20. What did we accomplish?

At the beginning of the lab:

```text
Kubernetes
   ↓
New Relic
   ↓
Infrastructure visibility
```

After this lab:

```text
Kubernetes
   |
   +-------------------+
   |                   |
Infrastructure      Application
   |                   |
Nodes               demo-app
Pods                   |
Metrics             APM Agent
Events                  |
Logs                    ↓
   |                 New Relic
   +-------------------+
```

We now have two different levels of observability.

### Kubernetes level

```text
Node
Pod
Container
CPU
Memory
Restarts
Kubernetes Events
```

### Application level

```text
Transactions
Latency
Errors
Traces
Application performance
```

This is the important concept of Lab 2.

---

# Lab 2 — Success Criteria

The lab is complete when all of these are true:

```text
[✓] demo-app namespace exists

[✓] New Relic license secret exists

[✓] demo-app Deployment exists

[✓] Two demo-app Pods are Running

[✓] demo-app Service exists

[✓] /api/fast works

[✓] /api/slow works

[✓] /api/error generates an error

[✓] Traffic has been generated

[✓] demo-app appears under APM & Services

[✓] Transactions are visible

[✓] Slow requests are visible

[✓] Application errors are visible
```

## Key takeaway

> **Lab 1 gave New Relic visibility into Kubernetes. Lab 2 adds visibility into the application running inside Kubernetes.**

The combination gives us:

```text
Kubernetes + Application
        ↓
     New Relic
        ↓
Infrastructure + APM
```

This will become important in the next lab, where we connect **logs and Kubernetes events** with the application information we have just collected.
