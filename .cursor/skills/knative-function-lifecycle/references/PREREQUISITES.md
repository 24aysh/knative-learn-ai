# Prerequisites Reference

This document lists every prerequisite for the Knative Function lifecycle.

## Required CLI Tools

| Tool       | Purpose                           | Install                                                           |
|------------|-----------------------------------|-------------------------------------------------------------------|
| `func`     | Knative Function CLI              | https://knative.dev/docs/functions/install-func/                  |
| `kubectl`  | Kubernetes cluster management     | https://kubernetes.io/docs/tasks/tools/                           |
| `git`      | Version control                   | https://git-scm.com/downloads                                    |
| `docker`   | Container build & push            | https://docs.docker.com/get-docker/                               |

**Alternative:** `podman` can replace `docker`.

## Optional CLI Tools

| Tool   | Purpose                         | Install                                                             |
|--------|---------------------------------|---------------------------------------------------------------------|
| `kind` | Local Kubernetes clusters       | https://kind.sigs.k8s.io/docs/user/quick-start/#installation       |
| `gh`   | GitHub CLI (variables/secrets)  | https://cli.github.com/                                             |
| `jq`   | JSON parsing in scripts         | `sudo apt install jq` or `brew install jq`                          |
| `curl` | HTTP requests                   | Usually pre-installed on Linux/macOS                                |

## Kubernetes Cluster

- A running Kubernetes cluster (kind, minikube, GKE, EKS, etc.)
- `kubectl` configured with the correct context
- Nodes in `Ready` state

### Creating a kind cluster

```bash
kind create cluster --name func-ci
kubectl config current-context   # should show: kind-func-ci
kubectl get nodes
```

## Knative Serving

Knative Serving **must** be installed for `func deploy` to work.

### Install commands (v1.19.0)

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.19.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.19.0/serving-core.yaml
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.19.0/kourier.yaml

kubectl patch configmap/config-network \
  -n knative-serving --type merge \
  -p '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'

kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.19.0/serving-default-domain.yaml
```

### Verify

```bash
kubectl get pods -n knative-serving     # all Running/Completed
kubectl get pods -n kourier-system      # all Running/Completed
```

## GHCR (GitHub Container Registry)

- A GitHub PAT with `write:packages` and `read:packages` scopes
- For private repos also add `repo` scope
- Login: `echo $PAT | docker login ghcr.io -u <username> --password-stdin`

## GitHub Actions Configuration

### Variables (no https://, no trailing slash, no spaces/newlines)

| Variable           | Value                       |
|--------------------|-----------------------------|
| REGISTRY_LOGIN_URL | `ghcr.io`                   |
| REGISTRY_USERNAME  | `<github-username>`         |
| REGISTRY_URL       | `ghcr.io/<github-username>` |

### Secrets

| Secret            | Value                     |
|-------------------|---------------------------|
| REGISTRY_PASSWORD | GitHub PAT                |
| KUBECONFIG        | `kind get kubeconfig ...` |

## Self-Hosted Runner

Required when deploying to a local cluster (kind/minikube).

The runner machine must have: `func`, `kubectl`, `docker`, access to the cluster, correct kubeconfig.

## Tekton Pipelines (optional)

Only needed for `func deploy --remote`. Install from https://tekton.dev/docs/installation/pipelines/
