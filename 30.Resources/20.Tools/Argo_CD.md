# Argo CD

## What is Argo CD?

Argo CD is a GitOps Continuous Delivery tool for Kubernetes.

Its main idea is simple:

```text
Git
 ↓
desired state
 ↓
Argo CD
 ↓
Kubernetes / K3s
 ↓
running state
```

Instead of manually deploying Kubernetes manifests:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

we store them in Git.

Argo CD watches Git and keeps the Kubernetes cluster synchronized with it.

---

# Why Argo CD?

Without Argo CD:

```text
Engineer
   ↓
kubectl apply
   ↓
Kubernetes
```

The engineer is responsible for changing the cluster.

With Argo CD:

```text
Engineer
   ↓
git push
   ↓
Git
   ↓
Argo CD
   ↓
Kubernetes
```

Now Git describes what should be running.

This is the core GitOps idea:

> Change Git, not the cluster.

---

# Desired State vs Actual State

This is the most important concept.

Git contains the desired state:

```yaml
replicas: 1

image:
  ghcr.io/user/myapp:abc123
```

Kubernetes contains the actual state:

```text
Deployment
   ↓
ReplicaSet
   ↓
Pod
   ↓
Container
```

Argo CD compares them:

```text
Desired State        Actual State
     Git                 K3s
      │                   │
      └──── Argo CD ──────┘
```

If they match:

```text
Synced
```

If they are different:

```text
OutOfSync
```

---

# Sync and Health

These are two different things.

## Sync Status

```text
Synced
```

means:

```text
Git configuration
      =
Cluster configuration
```

`OutOfSync` means they differ.

## Health Status

```text
Healthy
```

means the Kubernetes resources are operating correctly.

For example:

```text
Synced + Healthy
```

is the normal state.

But synchronization and health are not the same thing.

The cluster can match Git while the application itself has a problem.

---

# Reconciliation

Argo CD continuously compares:

```text
Git
 ↓
What SHOULD exist?

vs

Kubernetes
 ↓
What DOES exist?
```

Then it tries to reconcile the difference.

Conceptually:

```text
Observe
   ↓
Compare
   ↓
Detect difference
   ↓
Reconcile
   ↓
Observe again
```

This reconciliation loop is one of the key ideas behind GitOps.

---

# Automatic Sync

Argo CD can synchronize changes automatically.

Example:

```text
deployment.yaml

image: myapp:abc123
        ↓
change in Git
        ↓
image: myapp:def456
```

Argo CD detects the Git change and updates Kubernetes.

```text
Git commit
    ↓
Argo CD detects change
    ↓
Deployment updated
    ↓
K3s creates new Pod
    ↓
old Pod removed
```

No manual `kubectl apply` is required.

---

# Self Heal

Self Heal protects the desired state.

Suppose Git says:

```text
replicas: 2
```

Someone manually changes Kubernetes:

```text
replicas: 1
```

Now:

```text
Git        Cluster
 2            1
      ↓
   OutOfSync
```

With Self Heal enabled, Argo CD restores the state defined in Git:

```text
Cluster → replicas: 2
```

This is why manual cluster changes should not be the normal workflow in GitOps.

---

# Prune

Prune handles resources that were removed from Git.

Example:

Git originally contains:

```text
deployment.yaml
service.yaml
ingress.yaml
```

Then `ingress.yaml` is deleted.

Without pruning, the Ingress may remain in Kubernetes.

With Prune enabled:

```text
Delete from Git
      ↓
Argo CD detects it
      ↓
Delete from Kubernetes
```

So Git remains the complete description of the desired state.

---

# Argo CD Application

An Argo CD `Application` connects:

```text
Git Repository
      +
Git Path
      +
Revision
      +
Kubernetes Cluster
      +
Namespace
```

Example:

```text
Repository:
myapp-deploy-lab

Revision:
main

Path:
k8s

Destination:
K3s

Namespace:
myapp
```

Conceptually:

```text
GitHub
└── myapp-deploy-lab
    └── k8s/
        ├── deployment.yaml
        ├── service.yaml
        └── ingress.yaml
             │
             ↓
          Argo CD
             │
             ↓
         namespace
           myapp
```

---

# Argo CD and GitHub Actions

Argo CD and GitHub Actions solve different problems.

GitHub Actions handles CI:

```text
Source Code
    ↓
Build
    ↓
Docker Image
    ↓
Registry
```

Argo CD handles GitOps CD:

```text
Git manifests
     ↓
Argo CD
     ↓
Kubernetes
```

Together:

```text
Developer
    ↓
git push
    ↓
GitHub Actions
    ↓
Build Docker image
    ↓
GHCR
    ↓
Update Kubernetes manifest in Git
    ↓
Argo CD
    ↓
K3s
    ↓
Application
```

So:

```text
GitHub Actions = build the artifact
Argo CD         = deploy desired state
```

---

# Why `latest` is a Problem

Suppose the manifest contains:

```yaml
image: ghcr.io/user/myapp:latest
```

GitHub Actions builds a new image and pushes another `latest`.

But the Git manifest still says:

```yaml
image: ghcr.io/user/myapp:latest
```

From Git's point of view:

```text
Before → latest
After  → latest

No Git change.
```

Argo CD may therefore have no new desired state to detect.

This weakens the GitOps model.

---

# Immutable Image Tags

A better approach is to use the Git commit SHA.

Example:

```yaml
image: ghcr.io/user/myapp:93a6231
```

Next version:

```yaml
image: ghcr.io/user/myapp:f72bd91
```

Now the deployment change is visible in Git:

```text
93a6231
   ↓
f72bd91
```

This gives us:

```text
Code commit
    ↓
Docker image SHA
    ↓
Kubernetes manifest
    ↓
Git history
```

We can see exactly which image should be running.

---

# GitOps Rollback

Traditional Kubernetes rollback could be:

```text
kubectl rollout undo
```

But with Argo CD + Self Heal this creates a problem.

Suppose:

```text
Git     = v2
Cluster = v1
```

Argo CD sees:

```text
OutOfSync
```

and may restore:

```text
Cluster = v2
```

because Git still says v2.

Therefore the GitOps rollback should normally change Git.

Example:

```text
git revert
    ↓
new Git commit
    ↓
CI builds image
    ↓
manifest changes
    ↓
Argo CD detects change
    ↓
K3s rollout
    ↓
previous application version
```

The important principle is:

> Roll back the desired state, not only the running state.

---

# Our Full Pipeline

Our lab eventually became:

```text
app.py
   ↓
git push
   ↓
GitHub Actions
   ↓
Docker build
   ↓
image:<commit-SHA>
   ↓
GHCR
   ↓
deployment.yaml updated
   ↓
Git commit
   ↓
Argo CD
   ↓
K3s Deployment
   ↓
ReplicaSet
   ↓
Pod
   ↓
Container
```

This separates responsibilities clearly:

```text
Git              → source of truth
GitHub Actions   → CI automation
Docker           → application artifact
GHCR             → artifact storage
Argo CD          → GitOps CD
K3s              → orchestration
```

---

# Important Lab Limitation

Our GitHub Actions workflow updates `deployment.yaml` and pushes the change back to the same `main` branch.

This works for the lab, but it can create this situation:

```text
Developer push
      ↓
GitHub Actions
      ↓
bot commit
      ↓
origin/main moves forward
      ↓
local repository is behind
```

Then the local repository may need:

```bash
git pull --rebase origin main
```

This is useful for learning the complete pipeline, but it also shows why production GitOps repository and branching strategies need more thought.

---

# Mental Model

Remember Argo CD like this:

```text
Git
│
│ "This is what I want."
│
↓
Argo CD
│
│ compare + reconcile
│
↓
Kubernetes
│
│ "This is what currently exists."
│
↓
Application
```

The key idea is not the Argo CD UI.

The key idea is:

> **Git describes the desired state, and Argo CD continuously works to make the cluster match it.**