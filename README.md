# php-tools-iac

Infrastructure as Code (IaC) repository used by **ArgoCD** to deploy the
[`docker.io/nauw/php-tools:latest`](https://hub.docker.com/r/nauw/php-tools)
Docker image into a dedicated Kubernetes namespace following GitOps best
practices.

---

## Table of Contents

1. [Repository structure](#repository-structure)
2. [Prerequisites](#prerequisites)
3. [Kubernetes manifests overview](#kubernetes-manifests-overview)
4. [Quick start – manual apply](#quick-start--manual-apply)
5. [ArgoCD setup](#argocd-setup)
   - [Register this repository in ArgoCD](#1-register-this-repository-in-argocd)
   - [Create the Application](#2-create-the-application)
   - [Verify the deployment](#3-verify-the-deployment)
6. [Accessing the application](#accessing-the-application)
7. [Configure a Git webhook](#configure-a-git-webhook)
8. [Customisation](#customisation)
9. [Best practices applied](#best-practices-applied)

---

## Repository structure

```
php-tools-iac/
├── argocd/
│   └── application.yaml      # ArgoCD Application CRD
└── manifests/
    ├── kustomization.yaml    # Kustomize entry-point (used by ArgoCD)
    ├── namespace.yaml        # Dedicated 'php-tools' namespace
    ├── deployment.yaml       # Deployment (2 replicas, rolling update, probes)
    ├── service.yaml          # NodePort Service on port 80 (nodePort: 30080)
    └── ingress.yaml          # (Not used - NodePort is configured instead)
```

---

## Prerequisites

| Tool | Minimum version | Notes |
|------|----------------|-------|
| Kubernetes cluster | 1.25+ | Minikube, kind, k3s or managed cloud cluster |
| kubectl | 1.25+ | Configured to talk to the cluster |
| ArgoCD | 2.8+ | Installed in the `argocd` namespace |

### Install ArgoCD (if not already installed)

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be ready
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=120s

# Retrieve the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Access the ArgoCD UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open https://localhost:8080 in your browser
```

---

## Kubernetes manifests overview

### `manifests/namespace.yaml`
Creates the isolated `php-tools` namespace so that resources do not pollute
the default namespace.

### `manifests/deployment.yaml`
Deploys `docker.io/nauw/php-tools:latest` with:
- **2 replicas** and a zero-downtime **RollingUpdate** strategy
- CPU/memory **resource requests and limits**
- HTTP **liveness** and **readiness** probes
- Apache exposed on container port **80**
- `imagePullPolicy: Always` to always fetch the latest image tag

### `manifests/service.yaml`
Exposes the pods internally on port **80** via a `ClusterIP` Service. The
application can then be published through the included Ingress resource or
temporarily accessed with `kubectl port-forward`.

### `manifests/kustomization.yaml`
Lists all manifests so ArgoCD (which uses Kustomize natively) can render them
as a single set of resources.

---

## Quick start – manual apply

If you want to apply the manifests without ArgoCD:

```bash
kubectl apply -k manifests/
```

To remove everything:

```bash
kubectl delete -k manifests/
```

---

## ArgoCD setup

### 1. Register this repository in ArgoCD

**Via the UI:**

1. Open the ArgoCD UI → **Settings** → **Repositories** → **Connect Repo**.
2. Choose connection method: **HTTPS**.
3. Fill in:
   - **Repository URL**: `https://github.com/nauw/php-tools-iac`
   - Leave **Username / Password** empty (public repo).
4. Click **Connect**.

**Via the CLI:**

```bash
argocd repo add https://github.com/nauw/php-tools-iac \
  --name php-tools-iac
```

For a **private** repository, add credentials:

```bash
argocd repo add https://github.com/nauw/php-tools-iac \
  --username <github-user> \
  --password <github-pat>
```

---

### 2. Create the Application

**Option A – apply the provided manifest (recommended):**

```bash
kubectl apply -f argocd/application.yaml
```

**Option B – ArgoCD CLI:**

```bash
argocd app create php-tools \
  --repo https://github.com/nauw/php-tools-iac \
  --path manifests \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace php-tools \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

**Option C – ArgoCD UI:**

1. Click **New App**.
2. Fill in the form:

   | Field | Value |
   |-------|-------|
   | Application Name | `php-tools` |
   | Project | `default` |
   | Sync Policy | `Automatic` |
   | Repository URL | `https://github.com/nauw/php-tools-iac` |
   | Revision | `HEAD` |
   | Path | `manifests` |
   | Cluster URL | `https://kubernetes.default.svc` |
   | Namespace | `php-tools` |

3. Enable **Prune resources** and **Self Heal**.
4. Click **Create**.

---

### 3. Verify the deployment

```bash
# Check ArgoCD Application status
argocd app get php-tools

# Check Kubernetes resources
kubectl get all -n php-tools

# Check pod logs
kubectl logs -l app.kubernetes.io/name=php-tools -n php-tools
```

The application should be **Synced** and **Healthy** within a few seconds.

---

## Accessing the application

The repository now exposes the workload through the included **Ingress** and a
backing `ClusterIP` Service.

### Using the Ingress

Add an entry for `php-tools.local` in your local hosts file that points to your
Ingress controller, then open:

```text
http://php-tools.local/
```

### Temporary local access

You can also port-forward the service directly:

```bash
kubectl port-forward -n php-tools svc/php-tools 8080:80
# Access at http://localhost:8080
```

---

## Configure a Git webhook

Webhooks allow GitHub to notify ArgoCD immediately when you push a commit,
instead of waiting for the default 3-minute polling interval.

### Step 1 – Expose ArgoCD server

ArgoCD must be reachable from GitHub. You can:

- Use a **LoadBalancer** service:
  ```bash
  kubectl patch svc argocd-server -n argocd \
    -p '{"spec": {"type": "LoadBalancer"}}'
  ```
- Or use an **Ingress** for the ArgoCD server with a public hostname.

Note the external IP/hostname:

```bash
kubectl get svc argocd-server -n argocd
```

### Step 2 – Generate a webhook secret

```bash
# Generate a random secret
WEBHOOK_SECRET=$(openssl rand -hex 32)
echo "Save this secret: $WEBHOOK_SECRET"
```

### Step 3 – Configure the secret in ArgoCD

```bash
kubectl -n argocd patch secret argocd-secret \
  --type merge \
  -p "{\"stringData\": {\"webhook.github.secret\": \"$WEBHOOK_SECRET\"}}"
```

### Step 4 – Add the webhook in GitHub

1. Go to your fork of this repository on GitHub.
2. Navigate to **Settings** → **Webhooks** → **Add webhook**.
3. Fill in:

   | Field | Value |
   |-------|-------|
   | Payload URL | `https://<argocd-hostname>/api/webhook` |
   | Content type | `application/json` |
   | Secret | `<WEBHOOK_SECRET from step 2>` |
   | Events | **Just the push event** |

4. Click **Add webhook**.

From now on every `git push` to this repository will trigger an immediate sync
in ArgoCD.

---

## Customisation

| What to change | File | Field |
|----------------|------|-------|
| Number of replicas | `manifests/deployment.yaml` | `spec.replicas` |
| Docker image tag | `manifests/deployment.yaml` | `spec.template.spec.containers[0].image` |
| CPU / memory limits | `manifests/deployment.yaml` | `spec.template.spec.containers[0].resources` |
| Exposed domain | `manifests/ingress.yaml` | `spec.rules[0].host` |
| Ingress class | `manifests/ingress.yaml` | `spec.ingressClassName` |
| ArgoCD repo URL | `argocd/application.yaml` | `spec.source.repoURL` |
| Target cluster | `argocd/application.yaml` | `spec.destination.server` |

---

## Best practices applied

- **Dedicated namespace** – resources are isolated from other workloads.
- **Kustomize** – native ArgoCD rendering; no Helm required for simple
  deployments.
- **Rolling update strategy** with `maxUnavailable: 0` – zero-downtime
  deployments.
- **Liveness & readiness probes** – Kubernetes only routes traffic to healthy
  pods.
- **Resource requests & limits** – prevents noisy-neighbour issues and enables
  the scheduler to make placement decisions.
- **Non-root security context** – reduces the blast radius of a container
  compromise.
- **Automated sync with prune & self-heal** – the cluster always converges to
  what is in Git.
- **Cascade deletion finalizer** – removing the ArgoCD Application also removes
  all owned Kubernetes resources.
- **Retry with exponential backoff** – transient sync failures are retried
  automatically.
