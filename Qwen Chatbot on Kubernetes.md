# Qwen Chatbot on Kubernetes — 

**Instructor:** Rahul Chaubey
**Platform:** minikube on EC2 (Ubuntu)
**Model:** Qwen2.5-0.5B-Instruct (CPU-only)
**Namespace:** `chatbot`


---

## Table of contents

- [Part 0 — Pre-session checklist (do this the day before)](#part-0--pre-session-checklist-do-this-the-day-before)
- [Part 1 — The 5-minute LLM primer](#part-1--the-5-minute-llm-primer)
- [Topic 1 — Introduction: what we are building and why](#topic-1--introduction-what-we-are-building-and-why)
- [Topic 2 — Docker: shipping a sealed build](#topic-2--docker-shipping-a-sealed-build)
- [Topic 3 — Namespace: your own area in a shared cluster](#topic-3--namespace-your-own-area-in-a-shared-cluster)
- [Topic 4 — Deployment: the standing policy](#topic-4--deployment-the-standing-policy)
- [Topic 5 — Services: the one address that never changes](#topic-5--services-the-one-address-that-never-changes)
- [Topic 6 — HPA: the capacity system](#topic-6--hpa-the-capacity-system)
- [Topic 7 — Putting it all together](#topic-7--putting-it-all-together)
- [Corrected manifests (full YAML)](#corrected-manifests-full-yaml)
- [Troubleshooting quick reference](#troubleshooting-quick-reference)
- [Interview questions](#interview-questions)
- [Teardown](#teardown)

---

## The running analogy — Hotstar during an IPL match

Use this consistently. Do not mix metaphors mid-session.

| Streaming concept | Our system |
|---|---|
| Hotstar app on your phone | Frontend pod (Flask proxy + UI) |
| Transcoding / streaming server doing the heavy work | Backend pod (Flask + Qwen2.5) |
| The single URL `hotstar.com` | Kubernetes Service |
| The platform that runs and restarts servers | Kubernetes |
| Capacity system that adds servers at toss time | HPA |
| A sealed app build shipped to every phone | Docker image |
| Separate product areas: sports, VOD, payments | Namespaces |
| Staging vs production | minikube vs EKS |

---

## Part 0 — Pre-session checklist (do this the day before)

Everything in this section must be **finished and verified before students join**. If you do any of it live, you will lose 20+ minutes to downloads.

### 0.1 EC2 instance

| Setting | Value |
|---|---|
| Instance type | **t3.2xlarge** (8 vCPU / 32 GB) |
| AMI | Ubuntu Server 22.04 or 24.04 LTS |
| Storage | 40 GB gp3 |
| Region | ap-south-1 |

Why 2xlarge and not xlarge: the HPA demo scales the backend to 3 pods, each wanting ~1.5 GB RAM and a full CPU core under inference. On an xlarge you will throttle and the demo looks broken.

**Security group:**

| Type | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| SSH | TCP | 22 | My IP | Connect to instance |
| Custom TCP | TCP | 8080 | 0.0.0.0/0 | Reach the chatbot via port-forward |

### 0.2 Install Docker

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```

**Now log out and log back in.** Do not use `newgrp docker` as your permanent fix — it only affects one shell, and the moment you open a second terminal during the session you will hit `permission denied on /var/run/docker.sock`.

Verify after reconnecting:

```bash
docker ps
```

### 0.3 Install minikube and kubectl

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
```

```bash
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
```

```bash
minikube version && kubectl version --client
```

### 0.4 Start minikube

**Never run minikube with `sudo`.** It creates the cluster under root's home and then `kubectl` as your normal user cannot find the kubeconfig. This is the single most common minikube failure and it is painful to unwind mid-session.

```bash
minikube start --driver=docker --memory=20480 --cpus=6
kubectl get nodes
```

### 0.5 Enable metrics-server — before anything else

```bash
minikube addons enable metrics-server
```

Wait 2–3 minutes, then confirm it is actually returning data:

```bash
kubectl top nodes
```

If this errors, wait longer. Do not proceed until it returns numbers. Without this, your HPA shows `<unknown>` all session.

### 0.6 Build both images inside minikube's Docker daemon

```bash
git clone https://github.com/Raj-pro/Qwen-Chatbot.git
cd Qwen-Chatbot

eval $(minikube docker-env)

docker build -t chatbot-backend:latest ./backend
docker build -t chatbot-frontend:latest ./frontend
```

The backend build downloads ~1.8 GB of Qwen model weights and bakes them into the image. **This takes 5–10 minutes.** Do it now, not live.

Note we are **not** using `minikube image load`. See Topic 2 for why.

Verify:

```bash
docker images | grep chatbot
```

You should see roughly a 2.5 GB backend and a 150 MB frontend.

### 0.7 Full dry run, then reset

```bash
kubectl apply -f k8s/
kubectl rollout status deployment/backend -n chatbot --timeout=300s
kubectl get pods -n chatbot
```

Test it end to end, then wipe it so students watch it get built:

```bash
kubectl delete namespace chatbot
```

### 0.8 Pre-session verification checklist

- [ ] EC2 running, SSH works, port 8080 open in security group
- [ ] `docker ps` works **without sudo** in a fresh terminal
- [ ] `minikube status` shows Running
- [ ] `kubectl top nodes` returns real numbers
- [ ] `docker images | grep chatbot` shows both images (after `eval $(minikube docker-env)`)
- [ ] Manifests in `k8s/` have `imagePullPolicy: IfNotPresent`
- [ ] Backend deployment has **`resources.requests.cpu`** set
- [ ] Dry run completed successfully, then namespace deleted
- [ ] Three terminals open and SSH'd in
- [ ] Browser tab open at `http://<EC2-PUBLIC-IP>:8080`

### 0.9 Terminal layout for the session

You need **three terminals**, all SSH'd into the EC2 box. Several commands block forever, so this is not optional.

| Terminal | Purpose |
|---|---|
| **T1** | Main working terminal — apply, get, delete |
| **T2** | Blocking watchers — `logs -f`, `get hpa -w` |
| **T3** | `port-forward` (runs the whole session, never closed) |

---

## Part 1 — The 5-minute LLM primer

Deliver this before Topic 1. Without it, students cannot reason about why a pod needs 2 GB RAM or why CPU spikes.

**An LLM is a next-word predictor.** It does not know answers, does not look things up, does not search the internet. Given some text, it guesses the most likely next word. It is the suggestion bar above your phone keyboard, trained on far more text and far better at it.

**Text becomes numbers first.** The sentence is chopped into tokens (roughly word-pieces) and each becomes a number. This is tokenization.

**The numbers pass through the weights.** They flow through the model's parameters — the 500 million numbers in `Qwen2.5-0.5B`. This is a very large pile of multiplication. Out comes a probability score for every possible next token. **This multiplication is why the pod burns CPU.**

**The loop runs for every single word.** The model picks the winner, appends it, and runs the entire thing again. A 100-word answer means roughly 130 full passes. This one fact explains:

- why the reply appears word by word
- why one user pins a CPU core for several seconds
- why 20 students at once will trigger your HPA

**What `Qwen2.5-0.5B-Instruct` means:**

| Part | Meaning |
|---|---|
| Qwen | Open-weights model family from Alibaba Cloud |
| 2.5 | Generation |
| 0.5B | 500 million parameters — small enough to run on plain CPU |
| Instruct | Fine-tuned to follow instructions and hold a conversation |

**Two things to say out loud so nobody is misled:**

1. **The model has no memory.** Each request is independent. If you want conversation history, the application must resend it. This is exactly why the backend pod is stateless — and why we can scale it horizontally at all.
2. **It can be confidently wrong.** 0.5B is small. Answers will be short and sometimes incorrect. Say this *before* a student asks it something and gets a bad reply.

**Bridge line into the rest of the session:**

> "The AI part is just a program that does heavy math. Once you accept that, everything else today is normal Kubernetes — package it, run it, expose it, scale it."

---

## Topic 1 — Introduction: what we are building and why

### The selling story

> Think about Hotstar on a normal Tuesday afternoon — a few thousand people watching old episodes. Now think about the IPL final. Same platform, but crores of people arrive within ten minutes of the toss.
>
> Hotstar cannot keep that many servers running all year for one night in May. And they cannot run one server and let everyone buffer. They need something that watches the crowd, adds capacity as it arrives, and gives it back when the match ends.

Then land it: **our chatbot has exactly the same shape, just smaller.** One backend pod serves a couple of people. Twenty students at once means a queue. Kubernetes is the thing that notices and reacts.

### Real-world use case — the HR chatbot

A company's internal HR chatbot: 5 questions/minute normally, 200/minute during appraisal season. The company does not want to:

- pay for 40 servers running 24/7 just in case
- manually restart the chatbot every time it crashes
- have a single point of failure

Kubernetes gives all three: scale on demand, automatic restart, load spread across replicas.

### Architecture

Open `qwen-chatbot-architecture.excalidraw` at excalidraw.com.

**Teaching tip:** do not show the finished diagram. Delete most of the boxes and reveal them as you go — browser and backend first, then service, then HPA. Students follow a diagram being built far better than a diagram already built. `Alt + drag` duplicates a box, useful when demoing "now HPA adds a second pod."

The request path is one straight line:

**Browser → frontend Service → frontend pod → backend Service → backend pod → Qwen → back the same way.**

Three things to call out while it is on screen:

**Why two Services.** The frontend Service is reachable from outside. The backend Service is `ClusterIP` — no external address at all. On Hotstar you connect to `hotstar.com`, never to a transcoding server directly. The heavy machine stays private.

**Why the frontend is a separate pod.** A student will ask; it looks redundant. Answer: the frontend is ~150 MB and near-zero CPU. The backend is ~2.5 GB and pins a core. Splitting them means we scale only the expensive half. As one pod, every extra copy of the UI would drag along another full copy of the model.

**Why HPA points at the backend only.** The UI never gets busy. The model always does.

### Suggested flow for the session

Run the working app once at the very start so students see the goal. Then tear it down and rebuild it one piece at a time. Every topic after that becomes "here is the piece we are adding, here is what breaks without it."

---

## Topic 2 — Docker: shipping a sealed build

### The selling story

> When Hotstar ships a new app version, they do not send you a list saying "install Java 17, then this library, then that codec." They send one sealed build with everything inside. Your phone, my phone, a phone in Delhi, a phone in Dubai — same build, same behaviour.
>
> Docker does that for server software. The image is the sealed build.

### Real-world use case

"It works on my machine but not on the server." Developer has Python 3.11, server has 3.8. Or a library version differs. Docker eliminates this — the image contains the exact Python version, exact libraries, exact model files.

### What is inside the backend image

- A Python base image
- The Qwen2.5-0.5B-Instruct model weights, baked in at build time
- The Flask application code (`app.py`)
- gunicorn as the production server

Because the weights are baked in, the pod needs **no internet at runtime** and **no Hugging Face token**. This is a deliberate design choice worth naming.

### The critical change: do not use `minikube image load`

The original project uses:

```bash
minikube image load chatbot-backend:latest    # DO NOT DO THIS LIVE
```

This tars up the entire 1.8+ GB image on the host, copies it into the minikube container, then imports it. On EC2 that is **3–6 minutes of a completely silent terminal**. No progress bar. Students will think it hung and someone will Ctrl+C.

Instead, build directly inside minikube's Docker daemon so there is nothing to copy:

```bash
eval $(minikube docker-env)
docker build -t chatbot-backend:latest ./backend
docker build -t chatbot-frontend:latest ./frontend
```

### The ten-second demo that teaches the concept

This is a better teaching moment than the load command, because it makes the mechanism visible:

```bash
# In a shell that has NOT been switched
docker images | grep chatbot        # empty — this is the host daemon

eval $(minikube docker-env)
docker images | grep chatbot        # there they are — this is minikube's daemon
```

Two lists, two daemons. Kubernetes only ever sees images in **minikube's** daemon, never yours.

**Caveat to state clearly:** `eval $(minikube docker-env)` applies only to the current shell. Open a new terminal and you are back on the host daemon. If a student's build "disappears," that is why.

### Live moment — rebuild and show layer caching

Since you pre-built in Part 0, running the build again during the session finishes in seconds:

```bash
docker build -t chatbot-backend:latest ./backend
```

> "Notice it finished instantly. Docker did not redo the 1.8 GB download — that layer has not changed, so it reused it. This is why CI pipelines are not slow after the first run."

### Live moment — show the size difference

```bash
docker images | grep chatbot
```

Roughly 2.5 GB backend vs 150 MB frontend. That single line of output justifies the whole architecture and sets up HPA perfectly:

> "When we scale up, which one do you want to copy — the 2.5 GB one or the 150 MB one? That is why we split them."

### `imagePullPolicy` — the number one minikube lab failure

The tag `:latest` makes Kubernetes default to `imagePullPolicy: Always`. It will ignore your local image, go to Docker Hub, find nothing, and give you `ImagePullBackOff`.

Every container spec must have:

```yaml
imagePullPolicy: IfNotPresent
```

### Mapping table

| Streaming concept | Docker concept |
|---|---|
| Sealed app build | Docker image |
| Producing the build | `docker build` |
| Where builds are stored | Image registry / local daemon |
| The app running on your phone | Container running from the image |

---

## Topic 3 — Namespace: your own area in a shared cluster

### The selling story

> Hotstar is not one application. Behind that one app are dozens of separate systems built by separate teams — live sports, the VOD catalogue, payments, recommendations, ads. They all run on the same shared Kubernetes infrastructure.
>
> The sports team has a service called `backend`. The payments team also has a service called `backend`. Thrown into the same space, they collide.
>
> A namespace gives each team its own labelled area inside the same cluster: `sports-live`, `vod-catalog`, `payments`. Same building, separate zones.

### Real-world use case

In a real company cluster there may be dozens of applications. Namespaces let each team have its own area so names do not clash, and cleanup is one command.

### Theory

A Namespace is the simplest Kubernetes resource — it groups everything else. Ours is called `chatbot`. Every Deployment, Service, and HPA declares it belongs there.

### The apply-order lesson

The first manifest is `00-namespace.yaml`. **Kubernetes does not decide to create the namespace first — `kubectl` applies files in alphabetical filename order.** The namespace goes first purely because of the `00-` prefix.

Say this explicitly:

> "The `00-` is not decoration. If that file were named `namespace.yaml`, it would sort after `deployment.yaml`, and every deployment would fail with 'namespace not found'. Numbered prefixes control apply order."

### Practical

```bash
kubectl apply -f k8s/
kubectl get pods,hpa -n chatbot
```

Stop typing `-n chatbot` fifty times — after the first few commands, set the default and tell students what you did:

```bash
kubectl config set-context --current --namespace=chatbot
```

### The misconception you must kill

Every batch assumes a namespace is a firewall. **It is not. A namespace separates names, not network traffic.** A pod in `chatbot` can freely reach a pod in `payments` today, with no configuration.

Prove it in fifteen seconds:

```bash
kubectl run test --image=nginx -n default
kubectl exec -it test -n default -- curl backend.chatbot:5001/health
kubectl delete pod test -n default
```

It works — from a completely different namespace.

Say the line plainly: **namespaces are for organisation and cleanup, not security.** Real traffic isolation is NetworkPolicy, a different object entirely.

### Not everything is namespaced

```bash
kubectl get nodes -n chatbot
```

Returns all nodes, ignoring the flag entirely — nodes are cluster-scoped. Same for PersistentVolumes and StorageClasses. One command, concept understood.

### The cleanup selling point

```bash
kubectl delete namespace chatbot
```

One command removes everything on that floor. Compare to deleting five resources individually and forgetting one.

**Warn them:** this takes 20–40 seconds and there is no undo. Also name the classic failure they will hit eventually — a namespace stuck in `Terminating` forever means a finalizer is waiting on something.

### Mapping table

| Streaming concept | Kubernetes concept |
|---|---|
| The whole Hotstar platform | The cluster |
| The sports team's own area | Namespace (`chatbot`) |
| "Show me only sports team stuff" | `kubectl get ... -n chatbot` |
| Decommission that product | `kubectl delete namespace chatbot` |

---

## Topic 4 — Deployment: the standing policy

### The selling story

Hotstar does not have an engineer manually starting servers. They declare a policy: "always keep N streaming servers healthy and running this exact build." If one crashes at 2 AM, the platform replaces it immediately — nobody is woken up.

A Deployment is exactly that standing policy. You never start a Pod by hand. You describe the desired state, and Kubernetes continuously enforces it.

### Real-world use case

The backend pod crashes at 2 AM from a memory spike. Without a Deployment, someone gets paged. With one, Kubernetes notices within seconds and starts a replacement with the identical image, config, and resources. No human involved.

### What our two Deployments specify

**`01-backend-deployment.yaml`** — the heavy one:

- image `chatbot-backend:latest`, `imagePullPolicy: IfNotPresent`
- container port 5001
- requests 1.5Gi memory **and 500m CPU**
- limits 2Gi memory, 1500m CPU
- liveness and readiness probes

**`02-frontend-deployment.yaml`** — the light one:

- image `chatbot-frontend:latest`, `imagePullPolicy: IfNotPresent`
- container port 5000
- requests 128Mi memory and 100m CPU

### The CPU request is mandatory — this is not optional

> **This is the single most important line in the whole runbook.**
>
> HPA calculates CPU utilisation as a **percentage of the CPU request**. No CPU request means there is no denominator, so the HPA reports `<unknown>/60%` and **will never scale**, no matter how hard you load it.
>
> If `resources.requests.cpu` is missing from the backend Deployment, Topic 6 will fail live.

Verify before the session:

```bash
kubectl get deployment backend -n chatbot -o jsonpath='{.spec.template.spec.containers[0].resources}' | python3 -m json.tool
```

### Practical — watch the hiring happen

```bash
kubectl apply -f k8s/

kubectl rollout status deployment/backend -n chatbot --timeout=300s
kubectl rollout status deployment/frontend -n chatbot --timeout=60s
```

**Explain the timeout difference.** Backend gets 300s, frontend 60s. The backend must load 500 million parameters from disk into RAM before it can answer anything. The frontend has nothing to load.

This is also worth naming as a real production concern: **slow-starting pods need generous readiness probe delays**, otherwise Kubernetes kills them mid-startup in a restart loop.

### Practical — watch the model load (T2)

```bash
kubectl logs -n chatbot deployment/backend -f
```

Students should see the Qwen model loading into memory, gunicorn workers starting, and the server becoming ready. This is the moment the abstract "500 million parameters" becomes concrete.

### Practical — self-healing demo (the strongest moment in Topic 4)

```bash
kubectl get pods -n chatbot
kubectl delete pod <backend-pod-name> -n chatbot
kubectl get pods -n chatbot
```

A new pod with a new name appears within seconds. Same image, same config, no human.

**Sharpen it — watch it happen rather than checking after:**

```bash
kubectl get pods -n chatbot -w
```

Then delete the pod in another terminal. Students see `Terminating` and `ContainerCreating` appear simultaneously.

**The question to ask the class, which sets up Topic 5:**

> "The new pod has a different name and a different IP. So how does the frontend still find it?"

Do not answer. Move to Topic 5.

### Mapping table

| Streaming concept | Kubernetes concept |
|---|---|
| "Always keep N servers running this build" | Deployment |
| One actual server running right now | Pod |
| Health checks on each server | Liveness / readiness probes |
| Capacity reserved per server | `resources.requests` / `limits` |
| Automatic replacement of a dead server | Self-healing (Deployment controller) |

---

## Topic 5 — Services: the one address that never changes

### The selling story

We created this problem ourselves in Topic 4. We deleted the backend pod and a new one appeared — **with a different IP**. If the frontend had memorised the old IP, it would now be talking to nothing.

> You type `hotstar.com`. You have no idea which server answers, and you do not care. Tomorrow it is a different machine with a different IP and you never notice.
>
> A Service is that stable address. It always knows which pods are currently healthy and forwards traffic to them.

### Real-world use case

This is the mechanism behind zero-downtime rolling updates. During a deploy, old pods terminate and new ones start — IPs change constantly. Users never notice, because they always talk to the same Service name.

### Two Services in this project

| | Backend | Frontend |
|---|---|---|
| Type | `ClusterIP` | `NodePort` |
| Reachable from | Inside cluster only | Outside the cluster |
| Service port | 5001 | 80 |
| Target (container) port | 5001 | **5000** |
| Address used by callers | `http://backend:5001` | `http://<node-ip>:<nodeport>` |

**Note the frontend port mapping carefully.** The Flask app inside the container listens on **5000**, but the Service exposes **80**. That is why `port-forward ... 8080:80` works. Service port and container port are different things — this trips people up constantly.

### Practical

```bash
kubectl get svc -n chatbot
```

Point out: backend has a ClusterIP and `<none>` under EXTERNAL-IP. It is unreachable from outside by design.

### How the frontend finds the backend

The frontend never hardcodes a pod IP. It uses the Service name:

```python
BACKEND_URL = "http://backend:5001/chat"   # never changes
```

**Show the DNS explicitly** — this is what makes Services click:

```bash
kubectl exec -it deployment/frontend -n chatbot -- nslookup backend
```

Then explain the full name: `backend.chatbot.svc.cluster.local`. The short name `backend` only works because the caller is in the same namespace. From another namespace you would need `backend.chatbot`.

### Prove the Service survives pod death

This is the payoff for the Topic 4 cliffhanger:

```bash
# T1 — note the current endpoint IP
kubectl get endpoints backend -n chatbot

# delete the pod
kubectl delete pod <backend-pod-name> -n chatbot

# after the new pod is ready — different IP, same Service
kubectl get endpoints backend -n chatbot
```

The IP changed. The Service name did not. **That is the entire point of Services in one demo.**

### Reaching it from outside (T3 — leave this running)

```bash
kubectl port-forward --address 0.0.0.0 -n chatbot svc/frontend 8080:80
```

**`--address 0.0.0.0` is mandatory on EC2.** Without it, port-forward binds to `127.0.0.1` *inside the EC2 box* and your browser will never reach it — even with port 8080 open in the security group. This is the most common EC2 lab failure.

Then browse to `http://<EC2-PUBLIC-IP>:8080`.

On a local Mac instead:

```bash
kubectl port-forward -n chatbot svc/frontend 8080:80 &
```

### Mapping table

| Streaming concept | Kubernetes concept |
|---|---|
| `hotstar.com` — one stable address | Service DNS name |
| Internal server-to-server address | ClusterIP Service (backend) |
| Public entry point | NodePort Service (frontend) |
| Temporary tunnel for testing | `kubectl port-forward` |

---

## Topic 6 — HPA: the capacity system

### The selling story

Toss happens. Traffic goes vertical in ten minutes. Hotstar's capacity system watches load, adds servers automatically, and gives them back after the presentation ceremony. Nobody sits watching a dashboard with a finger on a button.

The HPA is that system. Our rule: **if average CPU across backend pods exceeds 60%, scale up to a maximum of 3. If it drops, scale back toward 1.**

### Real-world use case

The biggest cost-and-reliability lever for any production AI service. Near-zero traffic at 3 AM, a spike at 10 AM when offices open. Without autoscaling you either overpay all night or fall over at 10 AM.

### How the HPA decides

The HPA polls metrics-server every 15 seconds and asks: what percentage of its **requested** CPU is each backend pod using? It averages across pods and compares to the 60% target.

Restate the dependency chain, because it is where everything breaks:

**metrics-server must be running → backend must have a CPU request → only then does HPA produce a number.**

### Practical — confirm the dashboard is alive

```bash
kubectl top pods -n chatbot
kubectl get hpa -n chatbot
```

If TARGETS shows `<unknown>`, stop and fix it before continuing. Two possible causes:

1. metrics-server not ready — wait 2–3 minutes
2. backend Deployment has no `resources.requests.cpu` — this is a manifest bug, fix the YAML

### Practical — generating a real rush

> **The original lab's load generator does not work.** It hits `/health`, which returns a status string without touching the model. It costs almost no CPU. You would flood the backend with thousands of requests and watch CPU sit at 3%.
>
> Load must go through **actual inference** — `POST /chat`. And busybox `wget` cannot post JSON properly.

Use this instead — `curl` image, real inference, six concurrent loops, long prompt:

```bash
kubectl run load --image=curlimages/curl:latest --restart=Never -n chatbot -- \
  sh -c 'for i in 1 2 3 4 5 6; do (while true; do curl -s -X POST http://backend:5001/chat -H "Content-Type: application/json" -d "{\"message\":\"explain kubernetes networking in detail\"}" > /dev/null; done) & done; wait'
```

**Why six parallel loops:** each request is serialised — one loop waits for a full response before sending the next. One loop cannot saturate the backend. Six can.

**Why a long prompt:** more output tokens means more forward passes, which means more CPU. Ties directly back to the LLM primer.

### Practical — watch the scale-up (T2)

```bash
kubectl get hpa backend-hpa -n chatbot -w
```

And in a third view, the pods appearing:

```bash
kubectl get pods -n chatbot -w
```

Expected timeline:

| Time | What students see |
|---|---|
| 0:00 | Load pod starts |
| 0:15–0:45 | TARGETS climbs past 60% |
| 0:45–1:15 | REPLICAS goes 1 → 2 |
| 1:15–2:00 | New pod `ContainerCreating`, then Running |
| 2:00–3:00 | REPLICAS may reach 3 |

**Set expectations before you start:** the new pod takes 30–60 seconds to become Ready because it has to load the model. Say this out loud or the pause looks like a failure.

### Practical — ending the rush

```bash
kubectl delete pod load -n chatbot
```

> **Warn them about the wait.** The default HPA scale-down stabilisation window is **300 seconds**. For five full minutes after the load stops, absolutely nothing will happen. Students will think it broke.

Two options:

1. **Announce it and move on.** Explain that the delay is deliberate — it prevents thrashing when traffic is spiky. Come back to the terminal at the end of the session and show it settled at 1.
2. **Shorten it in the manifest** (recommended for a live session):

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 60
```

Even at 60 seconds, say why the delay exists. "Why not scale down instantly?" is a genuinely good interview question and this is the natural moment for it.

### Mapping table

| Streaming concept | Kubernetes concept |
|---|---|
| Capacity system watching load | HPA (`backend-hpa`) |
| The monitoring feed it reads | metrics-server |
| "Servers are working too hard" | CPU > 60% of request |
| Adding servers at toss time | Scale replicas up |
| Releasing them after the match | Scale replicas down |
| Budget ceiling on servers | `maxReplicas: 3` |

---

## Topic 7 — Putting it all together

### The selling story

A platform team does not ship a release and hope. They run a **smoke test** — one real request through the entire chain — before announcing it. That is exactly what we do now.

### The three-terminal layout

This is where the original lab breaks: `logs -f` and `port-forward` both block forever. Run them in separate terminals.

**T3 — port-forward (start first, leave running):**

```bash
kubectl port-forward --address 0.0.0.0 -n chatbot svc/frontend 8080:80
```

**T2 — live backend logs:**

```bash
kubectl logs -n chatbot deployment/backend -f
```

**T1 — everything else:**

```bash
kubectl get pods,svc,hpa -n chatbot
```

### The test order

```bash
curl -s -X POST http://localhost:8080/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "What is Kubernetes?"}' | python3 -m json.tool
```

**Run this while T2 is visible on screen.** Students see the request arrive in the backend logs at the same moment the answer appears in T1. That simultaneity is the single most convincing thing in the whole session.

### Trace the request out loud

Walk the path with the diagram on screen:

1. `curl` hits port 8080 on the EC2 box
2. `port-forward` tunnels it into the cluster, to the `frontend` Service on port 80
3. The Service forwards to a frontend pod on port 5000
4. Flask proxy posts to `http://backend:5001/chat`
5. Cluster DNS resolves `backend` to the backend Service ClusterIP
6. The Service forwards to whichever backend pod is healthy
7. Qwen runs ~130 forward passes generating the answer token by token
8. The response travels back along the identical path

**Then say the closing line:**

> "If this one command returns an answer, then the namespace, both Deployments, both Services, cluster DNS, the container images, and the model itself are all working. One command proves the entire system."

### Also test through the browser

Open `http://<EC2-PUBLIC-IP>:8080` and type a question. Watch T2 — the same log lines appear. This connects "the thing I built" to "the thing a user sees."

### The frontend proxy code (minimal, teachable)

```python
from flask import Flask, request, jsonify
import requests

app = Flask(__name__)

# Service name, never a pod IP
BACKEND_URL = "http://backend:5001/chat"

@app.route("/chat", methods=["POST"])
def chat():
    user_message = request.json.get("message", "")
    backend_response = requests.post(
        BACKEND_URL,
        json={"message": user_message},
        timeout=60
    )
    return jsonify(backend_response.json())

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Two lines worth pointing at:

- `BACKEND_URL` uses the **Service name**, not an IP. This is the entire Topic 5 lesson in one line.
- `host="0.0.0.0"` — binding to localhost inside a container makes it unreachable from outside the pod. A classic first-container mistake.

### Optional closing — the EKS bridge

Thirty seconds, high value:

> "Everything we ran today is standard Kubernetes. The exact same YAML runs on EKS in production. The only real change is the frontend Service type — `NodePort` becomes `LoadBalancer` or sits behind an Ingress. The Deployments, the Services, the HPA, the namespace — all identical. That is the point of Kubernetes: the same manifests run on a laptop and on a thousand-node cluster."

---

## Corrected manifests (full YAML)

Use these if the repo's manifests are missing the CPU request or `imagePullPolicy`.

### `00-namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: chatbot
```

### `01-backend-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: chatbot
spec:
  replicas: 1
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
          image: chatbot-backend:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5001
          resources:
            requests:
              memory: "1.5Gi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "1500m"
          readinessProbe:
            httpGet:
              path: /health
              port: 5001
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 12
          livenessProbe:
            httpGet:
              path: /health
              port: 5001
            initialDelaySeconds: 90
            periodSeconds: 20
            failureThreshold: 3
```

**Probe timings explained:** readiness starts checking at 30s with 12 retries, giving the model up to 150 seconds to load before the pod is marked unready. Liveness waits 90s so it never kills a pod that is still loading. Getting this wrong produces an endless restart loop — a genuinely common production bug.

### `02-frontend-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: chatbot
spec:
  replicas: 1
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
          image: chatbot-frontend:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5000
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /
              port: 5000
            initialDelaySeconds: 5
            periodSeconds: 5
```

### `03-services.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: chatbot
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 5001
      targetPort: 5001
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: chatbot
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 5000
```

### `04-hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: chatbot
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 1
  maxReplicas: 3
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
    scaleDown:
      stabilizationWindowSeconds: 60
```

---

## Troubleshooting quick reference

| Symptom | Cause | Fix |
|---|---|---|
| `ImagePullBackOff` | `imagePullPolicy: Always` with a local-only image | Set `imagePullPolicy: IfNotPresent`; confirm image exists after `eval $(minikube docker-env)` |
| `ErrImageNeverPull` | Image not in minikube's daemon | Run `eval $(minikube docker-env)` then rebuild |
| HPA TARGETS shows `<unknown>` | metrics-server not ready, **or** no CPU request on backend | Wait 2–3 min; verify `kubectl top pods`; add `resources.requests.cpu` |
| HPA never scales despite load | Load hitting `/health` instead of `/chat` | Use the `curlimages/curl` POST loop |
| Browser cannot reach `:8080` | port-forward bound to localhost | Add `--address 0.0.0.0`; check security group |
| `permission denied /var/run/docker.sock` | docker group not applied in this shell | Log out and back in (not just `newgrp`) |
| Pod stuck `Pending` | Not enough CPU/RAM on the node | Restart minikube with more resources |
| Backend restart loop | Liveness probe firing during model load | Raise `initialDelaySeconds` on liveness |
| `kubectl` cannot find cluster | minikube was started with `sudo` | `minikube delete`, restart without sudo |
| Second backend pod `Pending` during HPA demo | Node out of allocatable CPU | Reduce `requests.cpu`, or use a larger instance |
| Namespace stuck `Terminating` | A finalizer is waiting on a resource | Inspect with `kubectl get namespace chatbot -o yaml` |
| Chat replies are slow or poor quality | 0.5B model on CPU — expected | Set expectations up front |

---

## Interview questions

**Namespace**

1. Does a namespace provide network isolation between pods? *(No — names only. NetworkPolicy provides isolation.)*
2. Name three resources that are cluster-scoped rather than namespaced.
3. What does it mean when a namespace is stuck in `Terminating`?

**Deployment**

4. What is the difference between a Pod, a ReplicaSet, and a Deployment?
5. What happens if you set a `limit` but no `request`? *(Kubernetes defaults the request to the limit.)*
6. Why would a pod enter a restart loop even though the application is healthy? *(Liveness probe firing during a slow startup.)*
7. What is the difference between a liveness and a readiness probe?

**Service**

8. Why can't the frontend just use the backend pod's IP?
9. What is the full DNS name of a Service, and when do you need the long form?
10. What is the difference between `port` and `targetPort`?
11. Why is `kubectl port-forward` unsuitable for production?

**HPA**

12. HPA shows `<unknown>` for TARGETS. Give two possible causes.
13. Why does HPA need `resources.requests.cpu` specifically?
14. Why does scale-down have a stabilisation window but scale-up does not?
15. What is the difference between HPA, VPA, and Cluster Autoscaler?
16. Your HPA is at max replicas and pods are `Pending`. What is happening and what would fix it?

**Docker / images**

17. Why doesn't minikube see an image you just built on the host?
18. Why is `:latest` a poor tag choice for production?
19. What is the trade-off between baking model weights into an image versus downloading at startup?

---

## Teardown

**End of session:**

```bash
# stop port-forward in T3 with Ctrl+C, then:
kubectl delete namespace chatbot
minikube stop
```

**End of day — stop the EC2 instance in the console.** A t3.2xlarge left running costs roughly ₹28/hour, about ₹20,000/month if forgotten.

**Full removal:**

```bash
minikube delete
```

Then terminate the EC2 instance if you no longer need it.

### Teardown checklist

- [ ] Namespace deleted
- [ ] minikube stopped or deleted
- [ ] **EC2 instance stopped or terminated**
- [ ] EBS volume removed if the instance was terminated

---

## Session timing guide

| Segment | Minutes |
|---|---|
| LLM primer | 5 |
| Topic 1 — intro and architecture | 15 |
| Topic 2 — Docker | 20 |
| Topic 3 — Namespace | 15 |
| Topic 4 — Deployment + self-healing demo | 30 |
| Topic 5 — Services + endpoint demo | 25 |
| Topic 6 — HPA + load demo | 35 |
| Topic 7 — end-to-end test | 15 |
| Q&A and buffer | 20 |
