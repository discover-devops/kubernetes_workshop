#  Table of Contents


### **"Prometheus in Kubernetes using Helm"**


---

## **Module 1: Setting the Stage** 
1.1 What are we building today? (Big picture diagram)
1.2 Who is this session for?
1.3 Prerequisites & tools needed (kubectl, eksctl, helm, AWS CLI)

---

## **Module 2: Kubernetes Cluster Setup on AWS EKS** 
2.1 What is EKS and why AWS?
2.2 Installing prerequisites (eksctl, kubectl, AWS CLI)
2.3 Creating a 2-node EKS cluster using `eksctl` with standard disk (gp2)
2.4 Verifying the cluster — nodes, namespaces, context

---

## **Module 3: What is Helm?** 
3.1 The problem Helm solves (why not plain YAML?)
3.2 Helm architecture — Charts, Repos, Releases
3.3 Installing Helm
3.4 Your first `helm` commands (search, install, list, uninstall)

---

## **Module 4: Deploy a Microservice Application** 
4.1 What is a microservice? (vs monolith — key differences)
4.2 Our sample app — what it does (a simple multi-service app)
4.3 Deploying the app on Kubernetes
4.4 Exposing the app via **Ingress** (with NGINX Ingress Controller)
4.5 Testing the app — hit the endpoints

---

## **Module 5: Why Do We Need Special Monitoring Tools?** 
5.1 How do we monitor a monolith? (simple, centralized)
5.2 What changes with microservices? (distributed, ephemeral, dynamic)
5.3 The observability pillars — Metrics, Logs, Traces
5.4 Monitoring tools landscape:
- **Prometheus + Grafana** → Metrics
- **EFK / ELK Stack** → Logs
- **Jaeger / Tempo** → Traces
5.5 Why Prometheus for today's session?

---

## **Module 6: Prometheus — Concepts** 
6.1 What is Prometheus?
6.2 How Prometheus works — Pull model explained
6.3 Key components — Prometheus Server, Exporters, AlertManager, PushGateway
6.4 PromQL basics — the query language
6.5 What are metrics types? (Counter, Gauge, Histogram, Summary)

---

## **Module 7: Deploy Prometheus on Kubernetes using Helm** 
7.1 The `kube-prometheus-stack` Helm chart (what's inside)
7.2 Adding the Prometheus community Helm repo
7.3 Installing Prometheus stack with Helm
7.4 Verify — Pods, Services, ConfigMaps
7.5 Accessing Prometheus UI (port-forward or ingress)

---

## **Module 8: Connect Prometheus to Our Microservice** 
8.1 What is a ServiceMonitor?
8.2 Instrumenting the app (or using an existing exporter)
8.3 Writing a ServiceMonitor for our app
8.4 Verify metrics are being scraped in Prometheus UI
8.5 Basic PromQL queries on our app metrics

---

## **Module 9: Grafana Dashboards** 
9.1 Grafana already comes with kube-prometheus-stack!
9.2 Accessing Grafana UI
9.3 Exploring built-in dashboards (Node, Pod, Cluster)
9.4 Importing a dashboard for our microservice

---

## **Module 10: Wrap-Up & Q&A** 
10.1 What we built today (recap diagram)
10.2 What's next? (AlertManager, Thanos, Loki)
10.3 Resources & references
10.4 Q&A

---
---

