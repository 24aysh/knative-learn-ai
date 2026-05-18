---
name: knative-function-lifecycle
description: >
  Guide through the complete lifecycle of a Knative Function — from project creation,
  local build and deploy, Kubernetes cluster setup, Knative Serving installation,
  GitHub Actions CI/CD with GHCR and self-hosted runners, to troubleshooting common
  deployment errors. Use this skill whenever the user asks about creating, building,
  deploying, configuring CI/CD for, or troubleshooting a Knative Function.
paths:
  - "func.yaml"
  - "function.go"
  - "function_test.go"
  - ".github/workflows/func-deploy.yaml"
---

# Knative Function Lifecycle

This skill guides an AI agent through every phase of a Knative Function's lifecycle:
scaffolding, building, deploying, setting up CI/CD with GitHub Actions, and
troubleshooting known errors.


---

## When to Use

- The user wants to **create** a new Knative Function project.
- The user wants to **build** or **deploy** a Function locally or via CI/CD.
- The user is setting up a **Kubernetes cluster** (kind/minikube) for Functions.
- The user needs to **install Knative Serving** on a cluster.
- The user wants to configure **GitHub Actions CI/CD** for their Function.
- The user is configuring **GHCR** (GitHub Container Registry) for image push/pull.
- The user needs a **self-hosted GitHub Actions runner** for local cluster deployments.
- The user encounters **deployment errors** (registry parse errors, image pull failures).
- The user wants to set up **Tekton remote builds**.


---

## 📖 References

Detailed reference documents are in the `references/` directory. Load these
on demand when the agent needs deeper context.

| File                          | Contents                                                    |
|-------------------------------|-------------------------------------------------------------|
| `references/PREREQUISITES.md`    | Full list of every tool, install link, cluster requirements, GHCR setup, GitHub Actions variable/secret format |
| `references/TROUBLESHOOTING.md`  | Step-by-step guided fixes for every check-script failure — CLI tools, cluster, Knative, registry, Git, Tekton  |

When a user reports an error or a check script fails:

1. Read the relevant section of `references/TROUBLESHOOTING.md`.
2. Present the **guided fix** to the user with the exact commands.
3. After the user applies the fix, re-run the check script to verify.

---

## Phase 1: Create a Function Project

Scaffold a new Function project using `func create`. The user must choose a language
and template.

```bash
# Create a Go HTTP Function (adjust language/template as needed)
mkdir -p <project-dir>
cd <project-dir>
func create <function-name> --language go --template http
cd <function-name>
```

### Key file: `func.yaml`

This is the central metadata file for every Function. It stores:

- Function name
- Runtime (go, python, node, etc.)
- Builder
- Registry
- Deployment info (populated after deploy)

**Always verify the scaffold worked:**

```bash
ls
cat func.yaml
```

---

## Phase 2: Build the Function Locally

Before any CI/CD setup, confirm the project builds locally. This validates the
Function code and builder.

```bash
func build --registry ghcr.io/<github-username>
```

> **Important:** If the local build succeeds, most problems later are about
> registry auth, Kubernetes access, or GitHub Actions config — not the Function code.

---

## Phase 3: Create a Kubernetes Cluster

For local development the target is typically a `kind` cluster.

```bash
kind create cluster --name func-ci
```

**Verify the cluster is ready:**

```bash
kubectl config current-context   # should show: kind-func-ci
kubectl get nodes
```

---

## Phase 4: Install Knative Serving

`func deploy` creates a Knative Service, so Knative Serving **must** be installed.

### 4a. Install CRDs and core

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.19.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.19.0/serving-core.yaml
```

### 4b. Install Kourier networking

```bash
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.19.0/kourier.yaml
```

### 4c. Patch config-network to use Kourier

```bash
kubectl patch configmap/config-network \
  -n knative-serving \
  --type merge \
  -p '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'
```

### 4d. Install default domain helper

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.19.0/serving-default-domain.yaml
```

### 4e. Verify

```bash
kubectl get pods -n knative-serving
kubectl get pods -n kourier-system
```

All pods should reach `Running` or `Completed` state.

---

## Phase 5: Deploy Once Manually

Before involving GitHub Actions, deploy locally to validate the entire chain:

```bash
func deploy --registry ghcr.io/<github-username>
```

This checks:
1. The Function can build.
2. The image can be pushed to the registry.
3. The cluster is reachable.
4. Knative Serving can create the service.

**Verify:**

```bash
kubectl get ksvc
kubectl get pods
func list
```

---

## Phase 6: Initialize Git and Push to GitHub

```bash
cd <function-project-path>
git init
git branch -M main
git add .
git commit -m "Add knative function"
```

Connect to a GitHub repository:

```bash
git remote add origin https://github.com/<github-username>/<repo-name>.git
git push -u origin main
```

If `origin` already exists:

```bash
git remote set-url origin https://github.com/<github-username>/<repo-name>.git
git push -u origin main
```

---

## Phase 7: Generate the GitHub Actions Workflow

The `func config ci` command is **feature-gated**. Enable it with the
`FUNC_ENABLE_CI_CONFIG` environment variable.

### Standard (GitHub-hosted runner)

```bash
FUNC_ENABLE_CI_CONFIG=true func config ci
```

### Self-hosted runner (required for local kind/minikube clusters)

```bash
FUNC_ENABLE_CI_CONFIG=true func config ci --self-hosted-runner --force
```

This generates `.github/workflows/func-deploy.yaml`.

**Inspect the workflow:**

```bash
cat .github/workflows/func-deploy.yaml
```

Key workflow settings:

```yaml
runs-on: self-hosted
```

```yaml
env:
  FUNC_VERBOSE: "true"
  FUNC_BUILDER: <builder>
  FUNC_REGISTRY: ${{ vars.REGISTRY_LOGIN_URL }}/${{ vars.REGISTRY_USERNAME }}
```

**Commit and push:**

```bash
git add .github/workflows/func-deploy.yaml
git commit -m "Add func deploy workflow"
git push
```

---

## Phase 8: Why a Self-Hosted Runner is Needed

A GitHub-hosted runner runs on GitHub infrastructure and **cannot** reach a local
`kind` cluster whose kubeconfig points to `https://127.0.0.1:<port>`.

```
GitHub-hosted runner  →  CANNOT reach local kind cluster
Self-hosted runner    →  CAN reach local kind cluster (same machine)
```

**Rule:** Always use `--self-hosted-runner` when deploying to kind or minikube.

---

## Phase 9: Configure the Self-Hosted Runner

In the GitHub repository:

```
Settings → Actions → Runners → New self-hosted runner
```

Select **Linux** and follow the displayed commands:

```bash
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64.tar.gz -L <runner-download-url>
tar xzf ./actions-runner-linux-x64.tar.gz
./config.sh --url https://github.com/<username>/<repo> --token <runner-token>
./run.sh
```

Keep `./run.sh` running.

### Runner prerequisites

The runner machine **must** have:
- `func` CLI
- `kubectl`
- Docker or Podman
- Access to the kind cluster
- The correct kubeconfig context

**Verify on the runner machine:**

```bash
func version
kubectl config current-context
kubectl get nodes
docker version
```

---

## Phase 10: Configure GHCR (GitHub Container Registry)

The workflow pushes images to GHCR. The image name follows the pattern:

```
ghcr.io/<github-username>/<function-name>:latest
```

### Create a GitHub PAT with these scopes:

- `write:packages`
- `read:packages`
- `repo` (only if the repository is private)

### Test the login locally:

```bash
echo "<YOUR_GITHUB_PAT>" | docker login ghcr.io -u <github-username> --password-stdin
```

---

## Phase 11: Configure GitHub Actions Variables

In the GitHub repository:

```
Settings → Secrets and variables → Actions → Variables
```

Create these **exactly** (no `https://`, no trailing slash, no spaces, no newlines):

| Variable             | Value                        |
|----------------------|------------------------------|
| REGISTRY_LOGIN_URL   | `ghcr.io`                    |
| REGISTRY_USERNAME    | `<github-username>`          |
| REGISTRY_URL         | `ghcr.io/<github-username>`  |

The workflow combines them as:

```
REGISTRY_LOGIN_URL + "/" + REGISTRY_USERNAME  →  ghcr.io/<github-username>
```

If using the GitHub CLI:

```bash
gh variable set REGISTRY_LOGIN_URL --body "ghcr.io"
gh variable set REGISTRY_USERNAME --body "<github-username>"
gh variable set REGISTRY_URL --body "ghcr.io/<github-username>"
```

---

## Phase 12: Configure GitHub Actions Secrets

In the GitHub repository:

```
Settings → Secrets and variables → Actions → Secrets
```

| Secret            | Value                          |
|-------------------|--------------------------------|
| REGISTRY_PASSWORD | Your GitHub PAT                |
| KUBECONFIG        | Full kind kubeconfig YAML      |

### Generate the kind kubeconfig:

```bash
kind get kubeconfig --name func-ci > /tmp/kind-func-ci-kubeconfig
cat /tmp/kind-func-ci-kubeconfig
```

Copy the **entire** kubeconfig YAML into the `KUBECONFIG` secret.

> This works for self-hosted runners because the runner is on the same machine as
> the kind cluster.

---

## Phase 13: Known Error — Bad Registry Variable (Newline in Value)

### Symptom

```
Error: invalid registry: cannot determine function image: could not parse reference: ghcr.io
/24aysh/temp:latest
```

### Root Cause

The registry value contains a **newline character**, producing:

```
ghcr.io\n/24aysh/temp:latest
```

instead of:

```
ghcr.io/24aysh/temp:latest
```

### Fix

Delete and recreate the GitHub Actions variables **exactly** — no newline, no spaces:

```bash
gh variable set REGISTRY_LOGIN_URL --body "ghcr.io"
gh variable set REGISTRY_USERNAME --body "<github-username>"
gh variable set REGISTRY_URL --body "ghcr.io/<github-username>"
```

Or recreate them in the GitHub UI, being careful not to press Enter at the end of
the value.

---

## Phase 14: Known Error — Cluster Cannot Pull the Image

### Symptom

```
your function image is unreachable
It is possible that your docker registry is private
```

The image exists in GHCR but the kind cluster cannot pull it.

### Fix Option A: Make the GHCR Package Public (easiest for testing)

In GitHub:

```
Profile → Packages → <function-name> → Package settings → Change visibility → Public
```

Then rerun the workflow.

### Fix Option B: Add a Pull Secret to the Cluster

For private images, create a Kubernetes image pull secret:

```bash
kubectl create secret docker-registry ghcr-pull-secret \
  --docker-server=ghcr.io \
  --docker-username=<github-username> \
  --docker-password='<YOUR_GITHUB_PAT>' \
  --namespace default
```

Patch the default service account:

```bash
kubectl patch serviceaccount default \
  -n default \
  -p '{"imagePullSecrets":[{"name":"ghcr-pull-secret"}]}'
```

**Verify:**

```bash
kubectl get secret ghcr-pull-secret -n default
kubectl get serviceaccount default -n default -o yaml
```

Expected service account output:

```yaml
imagePullSecrets:
- name: ghcr-pull-secret
```

---

## Phase 15: Run the GitHub Action

Once all configuration is in place, pushing to the workflow branch triggers the
action. You can also rerun manually:

```
Repository → Actions → Func Deploy → Re-run jobs
```

### Expected workflow steps:

1. Checkout code
2. Run tests
3. Setup Kubernetes context
4. Login to container registry
5. Install func CLI
6. Deploy function (`func deploy`)

The deploy step uses `FUNC_REGISTRY` from the workflow environment.

---

## Phase 16: Verify Deployment

After the workflow succeeds:

```bash
kubectl get ksvc
kubectl get pods
func list
```

### Describe the Function:

```bash
func describe --path <function-project-path>
```

### Invoke the Function:

```bash
func invoke --target remote --path <function-project-path>
```

If Knative shows a ready service, the deployment is successful.

---

## Phase 17: Final Working Setup Checklist

Ensure all of the following are in place:

- [ ] Function project committed to GitHub
- [ ] `.github/workflows/func-deploy.yaml` generated and pushed
- [ ] Self-hosted runner running on the same machine as the kind cluster
- [ ] GitHub Actions variables set correctly:
  - `REGISTRY_LOGIN_URL=ghcr.io`
  - `REGISTRY_USERNAME=<github-username>`
  - `REGISTRY_URL=ghcr.io/<github-username>`
- [ ] GitHub Actions secrets set:
  - `REGISTRY_PASSWORD` (GitHub PAT)
  - `KUBECONFIG` (kind kubeconfig YAML)
- [ ] GHCR access configured (public package OR pull secret)
- [ ] Knative Serving installed on the kind cluster
- [ ] Successful `func deploy` from GitHub Actions

---

## Phase 18: Common Failure Points

When something goes wrong, check these in order:

1. **GitHub-hosted runner cannot reach local kind** — Use `--self-hosted-runner`.
2. **Self-hosted runner is not running** — Ensure `./run.sh` is active.
3. **KUBECONFIG secret is missing or wrong** — Regenerate with `kind get kubeconfig`.
4. **GitHub Actions variables have spaces or newlines** — Delete and recreate them.
5. **GHCR token lacks package permissions** — Needs `write:packages` + `read:packages`.
6. **GHCR package is private with no pull secret** — Make public or add `imagePullSecrets`.
7. **Knative Serving is not installed** — Check `kubectl get pods -n knative-serving`.
8. **Workflow branch mismatch** — Verify the workflow trigger branch matches the push branch.

### Key rules:

- **Local cluster?** → Self-hosted runner is mandatory.
- **Registry format:** `ghcr.io` (no `https://`, no trailing slash).
- **Combined registry:** `ghcr.io/<user-or-org>`.
- **Private GHCR?** → Either make public or configure `imagePullSecrets`.

---

## Phase 19: Tekton Remote Build Variant

Instead of building on the runner, you can trigger a **remote build** on the
Kubernetes cluster using Tekton Pipelines.

### Flow

```
GitHub Actions → func deploy --remote → Tekton PipelineRun → build in cluster → push → deploy
```

### Generate the Tekton remote workflow:

```bash
FUNC_ENABLE_CI_CONFIG=true func config ci --remote --self-hosted-runner --force
```

The generated workflow sets:

```yaml
FUNC_REMOTE: "true"
```

### Prerequisites

The cluster must have **Tekton Pipelines** installed.

**Verify Tekton:**

```bash
kubectl get pods -n tekton-pipelines
kubectl get pipelineruns
```

> Use Tekton remote builds when the runner should **not** do the container build
> locally — the runner only triggers the remote build and deployment.
