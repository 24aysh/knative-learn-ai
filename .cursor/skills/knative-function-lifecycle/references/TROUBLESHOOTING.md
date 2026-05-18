# Troubleshooting Guide

Guided solutions for every check-script failure. Find your error and follow the fix.

---

## CLI Tool Failures

### `func` not found

```bash
# Install func CLI
# Option A: Go install
go install knative.dev/func/cmd/func@latest

# Option B: Download binary
curl -Lo func https://github.com/knative/func/releases/latest/download/func_linux_amd64
chmod +x func && sudo mv func /usr/local/bin/
```

### `kubectl` not found

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

### Docker daemon not running

```bash
# Linux systemd
sudo systemctl start docker
sudo systemctl enable docker

# Docker Desktop: Open the Docker Desktop application

# Verify
docker info
```

---

## Kubernetes Cluster Failures

### No active kubeconfig context

```bash
# List available contexts
kubectl config get-contexts

# Use an existing context
kubectl config use-context <context-name>

# Or create a new kind cluster
kind create cluster --name func-ci

# Verify
kubectl config current-context
```

### Cannot reach cluster API

```bash
# Check if kind container is running
docker ps | grep kindest

# If stopped, restart
docker start <container-id>

# Re-export kubeconfig
kind export kubeconfig --name func-ci

# For other clusters, check VPN/network
kubectl cluster-info
```

### Nodes NOT Ready

```bash
# Inspect the problem node
kubectl describe node <node-name>

# Common fix: wait for node to initialize
kubectl get nodes -w

# For kind: delete and recreate
kind delete cluster --name func-ci
kind create cluster --name func-ci
```

---

## Knative Serving Failures

### knative-serving namespace does not exist

Knative Serving is not installed. Install it:

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.19.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.19.0/serving-core.yaml
```

Wait for pods:

```bash
kubectl get pods -n knative-serving -w
```

### Unhealthy pods in knative-serving

```bash
# Identify the failing pod
kubectl get pods -n knative-serving
kubectl describe pod <pod-name> -n knative-serving
kubectl logs <pod-name> -n knative-serving

# Common fix: re-apply
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.19.0/serving-core.yaml
```

### Kourier not installed

```bash
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.19.0/kourier.yaml
kubectl get pods -n kourier-system -w
```

### Ingress class not configured

```bash
kubectl patch configmap/config-network \
  -n knative-serving --type merge \
  -p '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'
```

---

## Registry Failures

### No GHCR credentials

```bash
# Create a PAT at https://github.com/settings/tokens
# Scopes: write:packages, read:packages (+ repo if private)

echo "<YOUR_GITHUB_PAT>" | docker login ghcr.io -u <username> --password-stdin
```

### Error: invalid registry — newline in variable

**Symptom:** `could not parse reference: ghcr.io\n/user/func:latest`

**Fix:** Delete and recreate the GitHub Actions variables without newlines:

```bash
gh variable set REGISTRY_LOGIN_URL --body "ghcr.io"
gh variable set REGISTRY_USERNAME --body "<username>"
gh variable set REGISTRY_URL --body "ghcr.io/<username>"
```

### Error: function image is unreachable (private registry)

**Option A — Make package public:**

GitHub → Profile → Packages → select package → Package settings → Change visibility → Public

**Option B — Add pull secret:**

```bash
kubectl create secret docker-registry ghcr-pull-secret \
  --docker-server=ghcr.io \
  --docker-username=<username> \
  --docker-password='<PAT>' \
  --namespace default

kubectl patch serviceaccount default -n default \
  -p '{"imagePullSecrets":[{"name":"ghcr-pull-secret"}]}'
```

---

## Git / GitHub Actions Failures

### Not a git repository

```bash
git init
git branch -M main
git add .
git commit -m "Initial commit"
```

### No origin remote

```bash
git remote add origin https://github.com/<user>/<repo>.git
git push -u origin main
```

### Workflow not found

```bash
FUNC_ENABLE_CI_CONFIG=true func config ci --self-hosted-runner --force
git add .github/workflows/func-deploy.yaml
git commit -m "Add CI workflow"
git push
```

### Self-hosted runner not running

```bash
cd actions-runner
./run.sh
# Keep this terminal open
```

Verify the runner appears as "Idle" in: Repository → Settings → Actions → Runners

---

## Tekton Failures

### tekton-pipelines namespace missing

Only needed for `func deploy --remote`. Install:

```bash
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl get pods -n tekton-pipelines -w
```
