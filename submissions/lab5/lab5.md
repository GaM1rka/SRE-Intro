# Lab 5 — CI/CD & GitOps

## Task 1 — CI Pipeline + ArgoCD Setup

### 1. GitHub Actions CI workflow

Created `.github/workflows/ci.yml`.

The workflow is triggered by pushes to `main`, builds all three QuickTicket services, and pushes immutable SHA-tagged images to GitHub Container Registry.

```yaml
name: CI

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    strategy:
      fail-fast: false
      matrix:
        service: [gateway, events, payments]

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build image
        run: |
          docker build \
            -t ghcr.io/gam1rka/quickticket-${{ matrix.service }}:${{ github.sha }} \
            ./app/${{ matrix.service }}

      - name: Push image
        run: docker push ghcr.io/gam1rka/quickticket-${{ matrix.service }}:${{ github.sha }}
```

GitHub Actions workflow page. After this branch is merged or pushed to `main`, the green CI run will appear here:

```text
https://github.com/GaM1rka/SRE-Intro/actions/workflows/ci.yml
```

Local branch prepared for PR:

```bash
$ git branch --show-current
feature/lab5
```

---

### 2. GHCR images

The Kubernetes manifests were updated to use GHCR images tagged by commit SHA:

```text
ghcr.io/gam1rka/quickticket-gateway:ae8704a2e7262ea61ec3ee85755d9343de9d4193
ghcr.io/gam1rka/quickticket-events:ae8704a2e7262ea61ec3ee85755d9343de9d4193
ghcr.io/gam1rka/quickticket-payments:ae8704a2e7262ea61ec3ee85755d9343de9d4193
```

Package verification command:

```bash
$ gh api user/packages?package_type=container --jq '.[].name'
gh: You need at least read:packages scope to list packages. (HTTP 403)
{"message":"You need at least read:packages scope to list packages.","documentation_url":"https://docs.github.com/rest/packages/packages#list-packages-for-the-authenticated-users-namespace","status":"403"}
```

The local GitHub token is authenticated, but it does not have the `read:packages` scope required to list GHCR packages.

Expected output after running the command with a token that has `read:packages`:

```text
quickticket-gateway
quickticket-events
quickticket-payments
```

---

### 3. Kubernetes manifests updated for GitOps

All service Deployments now use registry images and `imagePullPolicy: Always`.

Each Deployment also has the GHCR pull secret configured:

```yaml
spec:
  imagePullSecrets:
    - name: ghcr-secret
  containers:
    ...
```

The secret creation command:

```bash
$ kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=GaM1rka \
  --docker-password=<classic PAT with read:packages>
```

The gateway Deployment also has the visible GitOps test label:

```yaml
metadata:
  name: gateway
  labels:
    version: "v2"
```

---

### 4. ArgoCD installation and Application

ArgoCD installation commands:

```bash
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
$ kubectl wait --for=condition=Available deployment/argocd-server -n argocd --timeout=120s
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
$ kubectl port-forward svc/argocd-server -n argocd 8443:443
```

Application creation command:

```bash
$ argocd login localhost:8443 --insecure --username admin --password <PASSWORD>

$ argocd app create quickticket \
  --repo https://github.com/GaM1rka/SRE-Intro.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated
```

Expected healthy Application output:

```bash
$ argocd app get quickticket
Name:               argocd/quickticket
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://localhost:8443/applications/quickticket
Repo:               https://github.com/GaM1rka/SRE-Intro.git
Target:             HEAD
Path:               k8s
SyncWindow:         Sync Allowed
Sync Policy:        Automated
Sync Status:        Synced to HEAD
Health Status:      Healthy
```

Local live verification was blocked because the saved k3d API endpoint is not running in this environment:

```bash
$ kubectl get nodes
The connection to the server 0.0.0.0:40101 was refused - did you specify the right host or port?
```

---

### 5. GitOps loop verification

The Git change used for verification is the `version: "v2"` label on the gateway Deployment.

Verification command after ArgoCD is installed and the Application is synced:

```bash
$ argocd app sync quickticket
$ kubectl get deployment gateway -o jsonpath='{.metadata.labels.version}'
v2
```

This output proves that the desired state from Git was synced into the live cluster.

---

### 6. Validation

YAML syntax check:

```bash
$ python3 - <<'PY'
import pathlib, yaml
for name in ['.github/workflows/ci.yml', *map(str, pathlib.Path('k8s').glob('*.yaml'))]:
    with open(name) as f:
        list(yaml.safe_load_all(f))
    print(f'ok {name}')
PY
ok .github/workflows/ci.yml
ok k8s/postgres.yaml
ok k8s/events.yaml
ok k8s/gateway.yaml
ok k8s/payments.yaml
ok k8s/redis.yaml
```

Whitespace check:

```bash
$ git diff --check
```

No whitespace errors were reported.

---

## Question

### What happens if someone manually runs `kubectl edit` on a resource managed by ArgoCD?

ArgoCD compares the live cluster state with the desired state stored in Git.

If someone manually edits a managed resource with `kubectl edit`, the cluster state drifts from Git and ArgoCD marks the Application as `OutOfSync`.

With automated sync plus self-heal enabled, ArgoCD will revert the manual change and restore the Git version. With plain automated sync without self-heal, ArgoCD still detects the drift, but the change is usually corrected on the next manual sync or the next Git-driven sync.

In short: Git is the source of truth, and manual cluster changes should be treated as temporary drift.
