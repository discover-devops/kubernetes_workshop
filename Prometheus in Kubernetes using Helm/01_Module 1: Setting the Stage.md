# Module 1: Setting the Stage

---

## 1.1 What Are We Building Today?

### The Big Picture

Before we type a single command, let us understand what we are going to build by the end of this session.

Imagine this:

You are a DevOps Engineer at a company. The development team has built an online shopping application. It has separate services for the product catalog, shopping cart, payment, shipping, and more. Your job is to deploy it on the cloud and make sure you can monitor every service so that if anything breaks, you know about it before the customer complains.

That is exactly what we are doing today.

---

### What We Will Build

Here is the full picture of what we will build step by step.

**Step 1 - Your Laptop**

You will use your Windows laptop with PuTTY to SSH into an EC2 instance on AWS. This EC2 instance is called a Jump Box. Think of it as your remote workstation sitting on the cloud.

**Step 2 - EC2 Jump Box**

This is where all the action happens. We will install the following tools on this machine:

- aws cli
- eksctl
- kubectl
- helm
- git

From this machine we will create and manage everything else.

**Step 3 - AWS EKS Cluster**

We will use eksctl to create a 2-node Kubernetes cluster on AWS EKS. EKS stands for Elastic Kubernetes Service. AWS manages the Kubernetes control plane for us. We manage the worker nodes.

**Step 4 - Deploy the Microservice Application**

We will deploy Google Online Boutique, which is a sample e-commerce application made up of 11 microservices. It includes services like frontend, cart, payment, shipping, product catalog, and more. Each service runs as a separate pod inside the Kubernetes cluster.

**Step 5 - Expose the Application Using Ingress**

We will install the NGINX Ingress Controller and expose the application to the outside world through a single entry point. Students will be able to open the application in their browser.

**Step 6 - Deploy Prometheus and Grafana Using Helm**

We will use Helm to deploy the kube-prometheus-stack. This includes Prometheus for collecting metrics and Grafana for visualizing them through dashboards.

**Step 7 - Monitor the Application**

We will connect Prometheus to our microservice application, write queries using PromQL, and explore the Grafana dashboards to see what is happening inside our cluster in real time.

---

### Our Journey Today

| Step | What We Do | Approx Time |
|------|-----------|-------------|
| 1 | Set up EC2 Jump Box and install tools | 15 minutes |
| 2 | Create 2-node EKS cluster using eksctl | 20 minutes |
| 3 | Understand Helm and install it | 10 minutes |
| 4 | Deploy Online Boutique microservice app | 15 minutes |
| 5 | Expose the app via NGINX Ingress | 10 minutes |
| 6 | Understand why we need Prometheus | 10 minutes |
| 7 | Deploy Prometheus and Grafana via Helm | 15 minutes |
| 8 | Connect Prometheus to the app and explore | 15 minutes |
| 9 | Grafana dashboards | 10 minutes |

---

## 1.2 Who Is This Session For?

This session is for you if any of these sound familiar:

- You have heard of Kubernetes but never worked on it deeply
- You do not know what Helm is
- You have no idea how Prometheus works
- You want real hands-on experience, not just theory

You do NOT need to know any of the following before starting:

- Prometheus or Grafana
- Helm
- How EKS is created internally
- What a microservice architecture looks like in production

You DO need the following:

- An AWS account with enough permissions
- PuTTY installed on your Windows laptop
- Willingness to ask questions. There are no stupid questions today.

---

## 1.3 Prerequisites and Tools Needed

### On Your Windows Laptop

You only need two things on your laptop for this session.

| Tool | Purpose |
|------|---------|
| PuTTY | To SSH into the EC2 Jump Box |
| Web Browser | To access Grafana and the app UI |

That is it. Everything else runs on the EC2 Jump Box in the cloud.

---

### On the EC2 Jump Box

We will install all of these together during the session. You do not need to install anything in advance.

| Tool | Purpose |
|------|---------|
| aws cli | To talk to AWS APIs from the terminal |
| eksctl | To create and manage the EKS cluster |
| kubectl | To talk to the Kubernetes cluster |
| helm | To deploy apps using Helm charts |
| git | To clone the application repository |

---

### AWS Account Permissions

Make sure your AWS account or IAM user has the following permissions before the session starts.

- AmazonEC2FullAccess
- AmazonEKSFullAccess
- AmazonVPCFullAccess
- IAMFullAccess
- CloudFormationFullAccess

Important: eksctl uses AWS CloudFormation under the hood to create the EKS cluster. Without CloudFormation permissions, the cluster creation will fail silently or throw errors. So make sure this permission is in place.

---

### Common Question from Students

Question: Can I run all of this from my Windows laptop directly instead of using the EC2 Jump Box?

Answer: Technically yes. But in a classroom with 10 to 15 people, everyone has a different Windows version, different Git Bash version, and different PATH configurations. We will end up spending 30 minutes fixing individual laptops instead of learning. The EC2 Jump Box gives everyone the exact same clean Linux environment. Every command we type will work the same way for everyone. Trust the process.

---

## Summary of Module 1

By the end of this module you should know:

- What we are building today and why
- The role of each component in our architecture
- What tools we need and where they will run
- What AWS permissions are required

In the next module we will log into AWS, launch our EC2 Jump Box, install all the tools, and create our 2-node EKS cluster.

---
