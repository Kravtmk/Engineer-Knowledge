# Argo CD Practice

## Goal

Build and verify a complete GitOps deployment:

```text
app.py
   ↓
git push
   ↓
GitHub Actions
   ↓
Docker Image :SHA
   ↓
GHCR
   ↓
Git Manifest
   ↓
Argo CD
   ↓
K3s
   ↓
MyApp
```

Final result:

```text
v1 → automatic v2 deploy → GitOps rollback → v1
```

---

# Prerequisites

Before starting:

```text
K3s cluster        ✅
kubectl            ✅
Git repository     ✅
Dockerfile         ✅
GitHub Actions CI  ✅
GHCR               ✅
K8s manifests      ✅
```

Repository:

```text
myapp-deploy-lab/
├── app.py
├── Dockerfile
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── .github/
    └── workflows/
        └── build.yml
```

---

# Level 1 — Install Argo CD

Create namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd \
  --server-side \
  --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check:

```bash
kubectl get pods -n argocd
```

Wait until the main components are running:

```text
argocd-server
argocd-repo-server
argocd-application-controller
argocd-applicationset-controller
argocd-redis
```

### Checkpoint

- [ ] `argocd` namespace exists
- [ ] Argo CD Pods are Running

> For this lab `stable` is acceptable. For a controlled environment, pin a specific Argo CD version.

---

# Level 2 — Access Argo CD UI

For the lab, use port-forward instead of configuring Ingress/TLS.

On the K3s node:

```bash
kubectl port-forward svc/argocd-server \
  -n argocd \
  8080:443 \
  --address 0.0.0.0
```

Keep this terminal open.

Check:

```bash
sudo ss -lntp | grep 8080
```

Test:

```bash
curl -k https://NODE_IP:8080
```

Open:

```text
https://NODE_IP:8080
```

A certificate warning is expected in this lab.

## If UFW blocks access

Check:

```bash
sudo ufw status
```

Allow only the local LAN:

```bash
sudo ufw allow from 192.168.8.0/24 to any port 8080 proto tcp
```

Do not expose the lab UI to the whole network unnecessarily.

---

# Level 3 — Login

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Login:

```text
Username: admin
Password: <command output>
```

### Checkpoint

- [ ] Argo CD UI opens
- [ ] Login works
- [ ] Applications page is available

---

# Level 4 — Put Kubernetes State in Git

Argo CD needs the desired Kubernetes state in Git.

Create:

```bash
mkdir -p k8s
```

The repository should contain:

```text
myapp-deploy-lab/
├── app.py
├── Dockerfile
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── .github/
    └── workflows/
        └── build.yml
```

Check the image:

```bash
grep -A2 "image:" k8s/deployment.yaml
```

Initial lab example:

```yaml
image: ghcr.io/USER/myapp-deploy-lab:latest
imagePullPolicy: Always
```

Commit:

```bash
git add k8s/
git commit -m "Add Kubernetes manifests"
git push
```

### Understand

We now have:

```text
Git
└── k8s/
    ├── Deployment
    ├── Service
    └── Ingress
```

Git can now describe what should exist in K3s.

---

# Level 5 — Create Argo CD Application

In Argo CD create a new Application.

Example:

```text
Application Name: myapp
Project:          default

Sync Policy:      Automatic

Repository URL:   <your repository>
Revision:         main
Path:             k8s

Cluster URL:      https://kubernetes.default.svc
Namespace:        myapp
```

Enable:

```text
Prune Resources    ON
Self Heal          ON
```

Create the Application.

Check:

```bash
kubectl get applications -n argocd
```

Target state:

```text
myapp   Synced   Healthy
```

Check application Pods:

```bash
kubectl get pods -n myapp
```

### Checkpoint

- [ ] Application exists
- [ ] Status is `Synced`
- [ ] Health is `Healthy`
- [ ] MyApp Pod is Running

We now have:

```text
Git = desired state
        ↓
     Argo CD
        ↓
       K3s
```

---

# Level 6 — Understand the `latest` Problem

Before continuing, inspect:

```yaml
image: ghcr.io/USER/myapp-deploy-lab:latest
```

Imagine GitHub Actions builds a new image:

```text
old image → latest
new image → latest
```

But Git still contains:

```text
latest → latest
```

There may be no Git change for Argo CD to detect.

For GitOps, use an immutable image version:

```text
Docker image = Git commit SHA
```

Example:

```yaml
image: ghcr.io/USER/myapp-deploy-lab:93a6231
```

Then every deployment becomes visible in Git.

---

# Level 7 — Connect CI to GitOps CD

Edit:

```text
.github/workflows/build.yml
```

The workflow needs permission to push the manifest update:

```yaml
permissions:
  contents: write
  packages: write
```

After building and pushing the Docker image, update the Kubernetes image tag:

```yaml
- name: Update Kubernetes image tag
  run: |
    sed -i "s|image: ghcr.io/USER/myapp-deploy-lab:.*|image: ghcr.io/USER/myapp-deploy-lab:${{ github.sha }}|" k8s/deployment.yaml
```

Commit the new desired state:

```yaml
- name: Commit Kubernetes manifest
  run: |
    git config user.name "github-actions[bot]"
    git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

    git add k8s/deployment.yaml
    git commit -m "Deploy ${{ github.sha }}" || exit 0
    git push
```

Commit:

```bash
git add .github/workflows/build.yml
git commit -m "Add GitOps auto deploy"
git push
```

Now:

```text
git push
   ↓
GitHub Actions
   ↓
build image :SHA
   ↓
GHCR
   ↓
update deployment.yaml
   ↓
bot commit
   ↓
Argo CD
   ↓
K3s
```

---

# Level 8 — Automatic v1 → v2

Change the application:

```text
MyApp Docker v1
        ↓
MyApp Docker v2
```

Commit:

```bash
git add app.py
git commit -m "Release v2"
git push
```

Do not manually deploy anything to K3s.

Check CI:

```bash
gh run list
```

Watch the cluster:

```bash
kubectl get pods -n myapp -w
```

Expected:

```text
old Pod Running
      ↓
new Pod created
      ↓
new Pod Running
      ↓
old Pod Terminating
```

Check Argo CD:

```bash
kubectl get applications -n argocd
```

Expected:

```text
Synced
Healthy
```

Test application:

```bash
curl -H "Host: myapp.infrahome.dev" http://NODE_IP
```

Expected:

```html
<h1>MyApp Docker v2</h1>
```

### Checkpoint

- [ ] GitHub Actions succeeded
- [ ] New SHA exists
- [ ] Manifest was updated
- [ ] Argo CD synchronized it
- [ ] New Pod is Running
- [ ] Application returns v2

---

# Level 9 — GitOps Rollback

Do NOT use the normal Kubernetes rollback for this exercise:

```bash
kubectl rollout undo deployment/myapp -n myapp
```

Why?

```text
Git     = v2
K3s     = v1
          ↓
       OutOfSync
          ↓
       Self Heal
          ↓
K3s     = v2
```

Git remains the source of truth.

Find the v2 commit:

```bash
git log --oneline -5
```

Revert it:

```bash
git revert <RELEASE_V2_COMMIT>
```

Push:

```bash
git push
```

Then observe:

```text
git revert
    ↓
GitHub Actions
    ↓
build new image with v1 code
    ↓
GHCR :new-SHA
    ↓
deployment.yaml updated
    ↓
Argo CD
    ↓
K3s rolling update
    ↓
v1
```

Watch:

```bash
gh run list
```

and:

```bash
kubectl get pods -n myapp -w
```

Verify:

```bash
curl -H "Host: myapp.infrahome.dev" http://NODE_IP
```

Expected:

```html
<h1>MyApp Docker v1</h1>
```

Finally:

```bash
kubectl get applications -n argocd
```

Expected:

```text
Synced
Healthy
```

---

# Level 10 — Handle Bot Commit Conflict

Our lab workflow commits `deployment.yaml` back to `main`.

This means GitHub Actions can move `origin/main` forward while your local repository still has an older history.

Example:

```text
origin/main:

A ── B ── bot-commit


local:

A ── B ── Release-v2
```

A normal push may then be rejected.

Update your branch:

```bash
git pull --rebase origin main
```

Then:

```bash
git push
```

Result:

```text
A ── B ── bot-commit ── Release-v2
```

Do not solve this lab situation with:

```bash
git push --force
```

### Lesson

The lab works, but letting CI modify the same branch developers work on introduces Git coordination problems.

That is an architectural limitation worth remembering.

---

# Troubleshooting Challenge

If a deployment does not happen, troubleshoot layer by layer.

## CI

```bash
gh run list
```

## Git

Check whether `deployment.yaml` contains the expected SHA:

```bash
grep "image:" k8s/deployment.yaml
```

## Argo CD

```bash
kubectl get applications -n argocd
```

## Deployment

```bash
kubectl get deployment -n myapp
```

## Pods

```bash
kubectl get pods -n myapp
```

## Pod Details

```bash
kubectl describe pod <POD_NAME> -n myapp
```

## Logs

```bash
kubectl logs <POD_NAME> -n myapp
```

Follow the pipeline:

```text
Code
 ↓
GitHub Actions
 ↓
GHCR
 ↓
Git Manifest
 ↓
Argo CD
 ↓
Deployment
 ↓
Pod
 ↓
Application
```

Do not jump randomly between layers.

---

# Final Challenge 🥷

Repeat the lab from scratch without copying the solution step by step.

You should be able to:

- [ ] Install Argo CD
- [ ] Access the UI
- [ ] Create an Application
- [ ] Explain `Synced` vs `Healthy`
- [ ] Enable Automatic Sync
- [ ] Explain Prune
- [ ] Explain Self Heal
- [ ] Store Kubernetes manifests in Git
- [ ] Build an image tagged with commit SHA
- [ ] Update the desired state through CI
- [ ] Deploy v1 → v2 with one `git push`
- [ ] Observe the rolling update
- [ ] Roll back through Git
- [ ] Explain why `kubectl rollout undo` conflicts with GitOps
- [ ] Diagnose a failed deployment layer by layer

## Final Architecture

```text
Developer
    │
    │ git push
    ▼
GitHub
    │
    ▼
GitHub Actions
    │
    ├── Build Docker Image
    │
    ├── Tag :commit-SHA
    │
    ├── Push → GHCR
    │
    └── Update deployment.yaml
                │
                ▼
               Git
                │
                ▼
             Argo CD
                │
                ▼
               K3s
                │
                ▼
             MyApp
```

## I understand this lab when I can explain:

```text
Why Git is the source of truth
Why Argo CD needs a Git change
Why SHA is better than latest
Why Self Heal can undo manual changes
Why rollback should go through Git
Where CI ends and GitOps CD begins
```