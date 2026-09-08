# Argo CD

Quick reference for everyday Argo CD work.

> Git = desired state  
> Argo CD = keeps the cluster synchronized with Git

---

## Check Argo CD

```bash
kubectl get pods -n argocd
```

Check Applications:

```bash
kubectl get applications -n argocd
```

Healthy state:

```text
Synced
Healthy
```

---

## Check MyApp

```bash
kubectl get applications -n argocd
kubectl get pods -n myapp
kubectl get deployment -n myapp
```

Watch Pod changes:

```bash
kubectl get pods -n myapp -w
```

---

## Application Details

```bash
kubectl describe application myapp -n argocd
```

Use when:

```text
OutOfSync
Degraded
Unknown
```

---

## Argo CD UI — Lab Access

Start port-forward on the K3s node:

```bash
kubectl port-forward svc/argocd-server \
  -n argocd \
  8080:443 \
  --address 0.0.0.0
```

Open:

```text
https://NODE_IP:8080
```

Check listening port:

```bash
sudo ss -lntp | grep 8080
```

Test:

```bash
curl -k https://NODE_IP:8080
```

If UFW blocks access from the LAN:

```bash
sudo ufw allow from 192.168.8.0/24 to any port 8080 proto tcp
```

---

## Initial Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Login:

```text
Username: admin
Password: <output>
```

---

## GitOps Deploy

Normal workflow:

```text
Git change
    ↓
Argo CD
    ↓
K3s
```

In our CI/CD pipeline:

```text
git push
   ↓
GitHub Actions
   ↓
Docker image :SHA
   ↓
GHCR
   ↓
deployment.yaml updated
   ↓
Argo CD
   ↓
K3s rollout
```

Check after deployment:

```bash
kubectl get applications -n argocd
kubectl get pods -n myapp
```

Expected:

```text
Synced
Healthy
```

---

## Image Version

Prefer immutable image tags:

```yaml
image: ghcr.io/USER/APP:<commit-SHA>
```

Avoid using only:

```yaml
image: ghcr.io/USER/APP:latest
```

For GitOps, a new SHA creates a visible change in Git:

```text
old SHA → new SHA
        ↓
Argo CD detects change
```

---

## Rollback

Git is the source of truth.

Check history:

```bash
git log --oneline
```

Rollback a change:

```bash
git revert <COMMIT_HASH>
git push
```

Then verify:

```bash
kubectl get applications -n argocd
kubectl get pods -n myapp -w
```

Avoid using this as the normal GitOps rollback:

```bash
kubectl rollout undo deployment/myapp -n myapp
```

With Self Heal enabled:

```text
Git = v2
Cluster = v1
     ↓
OutOfSync
     ↓
Argo CD
     ↓
Cluster = v2 again
```

Change Git instead.

---

## Troubleshooting

### 1. Check Argo CD

```bash
kubectl get applications -n argocd
```

### 2. Check Application

```bash
kubectl describe application myapp -n argocd
```

### 3. Check Deployment

```bash
kubectl get deployment -n myapp
```

### 4. Check Pods

```bash
kubectl get pods -n myapp
```

### 5. Describe Problem Pod

```bash
kubectl describe pod <POD_NAME> -n myapp
```

### 6. Check Logs

```bash
kubectl logs <POD_NAME> -n myapp
```

Follow the chain:

```text
Git
 ↓
Argo CD
 ↓
Deployment
 ↓
Pod
 ↓
Logs
```

---

## Status

```text
Synced      = cluster matches Git
OutOfSync   = cluster differs from Git

Healthy     = application resources are healthy
Degraded    = something is not working correctly
```

---

## Remember

```text
Git     = source of truth
Argo CD = synchronization / reconciliation
K3s     = running state
```

Normal operation:

```text
Change Git
    ↓
Argo CD detects it
    ↓
Argo CD changes K3s
```

Not:

```text
Change K3s manually
    ↓
fight with Argo CD 😄
```

Full installation, configuration, v1 → v2 deployment and rollback lab:

```text
35.Practice/20.Tools/Argo_CD.md
```