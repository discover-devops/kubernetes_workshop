# Kubernetes API Request Lifecycle — Authentication, Authorization & Admission Control

**Build Automate Architect**
*Context → Concept → Lab*

---

## Where Does Your YAML Actually Go?

When you run:

```bash
kubectl apply -f deployment.yaml
```

The request goes to one Kubernetes component: the **kube-apiserver**.

But the API Server never blindly accepts a request. Before anything is stored in `etcd`, it passes through **three distinct security and validation phases**:

```text
kubectl apply
      │
      ▼
API Server
      │
      ▼
Authentication
      │
      ▼
Authorization
      │
      ▼
Admission Controllers
      │
      ▼
Store in etcd
```

A common mix-up is treating Authentication, Authorization, and Admission Control as the same thing. They are not — each one asks a completely different question. Let's walk through each stage.

---

## Step 1: Authentication — "Who are you?"

Kubernetes first verifies the identity of the requester.

Examples of how identity is proven:

- Client Certificate
- Bearer Token
- Service Account
- OIDC Login

If identity cannot be verified, the request stops here:

```text
401 Unauthorized
```

**Analogy:** Imagine walking into an airport. Security asks, "Show me your passport." That's Authentication.

---

## Step 2: Authorization — "What are you allowed to do?"

Once Kubernetes knows who you are, the next question is what you're permitted to do.

For example: can this user create Pods? Delete Nodes? Create Secrets?

This is checked using:

- RBAC
- ABAC
- Webhook authorization
- Node Authorization

If permission is denied:

```text
403 Forbidden
```

**Analogy (continuing the airport):** Security already verified your passport. Now they ask, "Do you have a boarding pass?"

- Passport = Authentication
- Boarding Pass = Authorization

---

## Step 3: Admission Control — "Is this object acceptable?"

This is the stage most people misunderstand at first.

Authentication answered *who are you*. Authorization answered *are you allowed*. Admission Control asks a different question entirely:

> "Is this request valid according to cluster policies?"

Notice the shift — **it's no longer checking the user.** It's checking **the object being created.**

### Example: Mutation

You create a Pod:

```yaml
image: nginx
```

An Admission Controller automatically adds:

```yaml
imagePullPolicy: Always
```

You never wrote that line — Kubernetes inserted it. This is called **Mutation**.

### Example: Validation

Company policy requires every Pod to have:

```yaml
labels:
   owner:
```

You forgot to include it. An Admission Controller rejects the request. This is called **Validation**.

---

## Two Types of Admission Controllers

**1. Mutating Admission Controller** — changes the object.

You submit:
```yaml
containers:
- image: nginx
```

It gets changed to:
```yaml
containers:
- image: nginx
  imagePullPolicy: Always
```

**2. Validating Admission Controller** — never changes anything. It only responds:

```text
Allowed
```
or
```text
Rejected
```

---

## Real-World Example: A Company Like Netflix

Suppose developers at a large company are creating thousands of Pods every day, and company policy requires:

- Every Pod must have labels
- Every image must come from the internal registry
- No privileged containers
- Memory limit is mandatory
- CPU limit is mandatory

Developers will inevitably forget some of these rules from time to time. Admission Controllers are what enforce them automatically, protecting the cluster without relying on every developer remembering every rule.

---

## Where Exactly Does This Happen?

```text
kubectl apply
      │
      ▼
API Server
      │
Authentication
      │
Authorization
      │
Admission Controller
      │
etcd
```

The key detail: **the object has not yet reached `etcd`** at the Admission Controller stage. Admission Controllers sit *inside* the API Server's request pipeline, as the final checkpoint before anything is persisted.

---

## An Analogy: Entering a Corporate Office

- **Security Guard:** "Who are you?" → Authentication
- **Reception:** "Do you have permission to enter this floor?" → Authorization
- **Dress Code Checker:** "You're allowed inside, but remove your helmet," or "Formal shoes are mandatory." → Admission Controller

The dress code checker doesn't care who you are. It cares whether you comply with policy.

---

## Common Built-in Admission Controllers

A few examples (no need to memorize the full list):

- NamespaceLifecycle
- LimitRanger
- ResourceQuota
- DefaultStorageClass
- DefaultTolerationSeconds
- PodSecurity
- MutatingAdmissionWebhook
- ValidatingAdmissionWebhook

The important takeaway: some controllers **add defaults**, while others **enforce policies**.

---

## The Complete Request Flow

```text
Developer

kubectl apply -f deployment.yaml
           │
           ▼
      kube-apiserver
           │
           ▼
Authentication
"Who are you?"
           │
           ▼
Authorization
"What are you allowed to do?"
           │
           ▼
Admission Controller
"Is this object allowed in my cluster?"
           │
           ▼
Store in etcd
           │
           ▼
Scheduler notices the new Pod
           │
           ▼
Pod gets created
```

### Memory Trick

- **Authentication** → Who are you?
- **Authorization** → What are you allowed to do?
- **Admission Control** → Is the resource itself acceptable according to cluster policies?

---

# Deep Dive: Where Do Admission Controllers Actually Come From?

There are **two categories** of Admission Controllers:

1. **Built-in Admission Controllers** — already part of Kubernetes
2. **Admission Webhooks** — custom policies written by you or your organization

---

## 1. Built-in Admission Controllers

These are written by the Kubernetes project itself and compiled directly into the `kube-apiserver` binary. When the API server starts, many of these controllers are enabled by default.

For example, **DefaultStorageClass** automatically assigns a default StorageClass if one isn't specified. This logic ships with Kubernetes — you don't write it yourself.

---

## 2. Custom Admission Controllers (Webhooks)

This is what most enterprises actually use for their own policies.

Suppose your company requires every Pod to have these labels:

- `owner`
- `environment`
- `cost-center`

Kubernetes has no built-in knowledge of your company's policies — so you write them yourself, as a small web application (commonly in Go, Python, or Java) that exposes an HTTPS endpoint, for example:

```text
https://policy.company.com/mutate
```
or
```text
https://policy.company.com/validate
```

Kubernetes calls this application whenever a matching resource (like a Pod) is created.

### Walking Through an Example

A developer submits:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  ...
```

No labels included.

Instead of immediately storing this, the API Server sends the Pod specification to your webhook:

```text
Developer → API Server → Admission Webhook
```

Your webhook receives JSON like:

```json
{
  "kind": "Pod",
  "metadata": {
      "name": "nginx"
  }
}
```

Your webhook logic checks for a missing label and, if absent, adds one — then returns a patch:

```json
{
   "patch": {
      "metadata": {
          "labels": {
              "owner": "platform-team"
          }
      }
   }
}
```

The API Server applies that patch. The object finally stored in `etcd` becomes:

```yaml
metadata:
  labels:
     owner: platform-team
```

— even though the developer never wrote that label themselves.

---

## How Does Kubernetes Know Your Webhook Exists?

You register it using two Kubernetes resources:

- **MutatingWebhookConfiguration**
- **ValidatingWebhookConfiguration**

For example, a `MutatingWebhookConfiguration` tells Kubernetes: whenever a Pod is created, call `https://policy.company.com/mutate`. This is how Kubernetes knows where to send every matching request.

---

## Complete Flow With Webhooks

```text
kubectl apply
        │
        ▼
API Server
        │
Authentication
        │
Authorization
        │
Mutating Webhook
(Add Labels)
        │
Validating Webhook
(Check Policies)
        │
Store in etcd
```

---

## Real Enterprise Example

Imagine working at a company like Oracle, where policy requires every Pod to have:

```yaml
labels:
   cost-center
   application
   owner
```

Rather than expecting 500 developers to remember this every time, the Platform Team writes a single webhook. Every Pod automatically gets these labels applied — developers don't even need to know it's happening.

### Another Example: Registry Enforcement

Company policy: "No Docker Hub images."

A developer writes:

```yaml
image: nginx
```

The webhook checks whether the image starts with `company-registry.com`. It doesn't, so the webhook returns:

```text
Rejected
```

The API Server returns an error to the developer:

```text
Error: Image must come from the internal registry.
```

---

## Who Writes These Webhooks?

Usually the **Platform Engineering**, **Cloud Platform**, or **DevOps** team — not application developers.

---

## An Easier Path: Policy Engines

Instead of building and maintaining a custom webhook server, many organizations use policy engines such as:

- **Kyverno**
- **OPA Gatekeeper**

These let you define policies in YAML (or a policy language) rather than writing a web server yourself. For example, in Kyverno:

```yaml
match:
  resources:
    kinds:
      - Pod

mutate:
  patchStrategicMerge:
    metadata:
      labels:
        owner: platform-team
```

You write a YAML policy, and Kyverno acts as the admission webhook on your behalf — no custom server required.

---

## The Big Picture

Admission Controllers work like company policy enforcement. Kubernetes ships with some built-in rules, but every organization has its own standards. Rather than modifying Kubernetes itself, a Platform Team registers a webhook with the API Server. Whenever a resource is created, Kubernetes pauses, calls the webhook, and asks: "Should I modify this object? Should I reject it?" The webhook replies, and only then is the resource stored in `etcd`.

---

## Recap

1. Every `kubectl apply` passes through **Authentication → Authorization → Admission Control** before reaching `etcd`.
2. Authentication checks identity. Authorization checks permissions. Admission Control checks the object itself.
3. Admission Controllers are either **Mutating** (change the object) or **Validating** (accept/reject it).
4. Admission Controllers come in two forms: **built-in** (shipped with Kubernetes) and **custom webhooks** (written by your organization, or via policy engines like Kyverno / OPA Gatekeeper).
