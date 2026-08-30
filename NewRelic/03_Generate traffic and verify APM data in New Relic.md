Next is **Step 11: Generate traffic and verify APM data in New Relic**.

But before that, we should make sure the app is reachable. We already know the Pods are running, so now:

### Step 1 — Start port-forward

On the EC2 instance, run:

```bash
kubectl port-forward \
  --address 0.0.0.0 \
  -n demo-app \
  svc/demo-app 8080:80
```

**Keep this terminal running.**

This gives us:

```text
Internet
   ↓
EC2_PUBLIC_IP:8080
   ↓
kubectl port-forward
   ↓
demo-app Service :80
   ↓
demo-app Pods :3000
```

### Step 2 — Allow port 8080 in EC2 Security Group

Add an inbound rule:

```text
Type:   Custom TCP
Port:   8080
Source: YOUR_PUBLIC_IP/32
```

Then from your laptop/browser:

```text
http://EC2_PUBLIC_IP:8080/api/fast
```

If that works, test:

```text
http://EC2_PUBLIC_IP:8080/api/slow
```

and:

```text
http://EC2_PUBLIC_IP:8080/api/error
```

### Step 3 — Generate demo traffic

Open **another EC2 terminal**:

```bash
for i in {1..50}; do
  curl -s http://127.0.0.1:8080/api/fast > /dev/null
  curl -s http://127.0.0.1:8080/api/slow > /dev/null
  curl -s http://127.0.0.1:8080/api/error > /dev/null || true
  sleep 1
done
```

This creates approximately:

```text
50 × /api/fast
50 × /api/slow
50 × /api/error
```

The purpose is simply to generate **application telemetry**.

### Step 4 — Check New Relic

Go to:

**New Relic → APM & Services**

Look for:

```text
demo-app
```

If it appears, open it and look for the transaction activity.

**Do these steps first. Don't move to Logs yet.**

The next thing after successful APM verification is **Lab 3: Logs + Kubernetes Events + correlation**.
