# MicroK8s Homelab Runbook

This repository is now organized as a runbook for bringing up a small
MicroK8s-based homelab stack in a sensible order. It is not a production
platform guide. The goal is to get a working cluster, install a core set of
services, and leave clear checkpoints between each phase.

## What this runbook covers

Core path:

1. Prepare the host and install MicroK8s
2. Enable the cluster features the rest of the stack depends on
3. Confirm ingress, DNS, metrics, and storage are healthy
4. Install Argo CD
5. Install Gitea
6. Expose an existing monitoring stack

Optional path:

1. Install Linkerd
2. Install Permission Manager
3. Add code-server

## Repository map

- [`argocd/README.md`](argocd/README.md)
- [`gitea/README.md`](gitea/README.md)
- [`prometheus/README.md`](prometheus/README.md)
- [`linkerd/README.md`](linkerd/README.md)
- [`permission-manager/README.md`](permission-manager/README.md)
- [`code-server/README.md`](code-server/README.md)

Reusable example files:

- [`examples/argocd/ingress-argocd.yaml`](examples/argocd/ingress-argocd.yaml)
- [`examples/gitea/gitea-values.yaml`](examples/gitea/gitea-values.yaml)
- `examples/linkerd/ingress-linkerd.yaml`
- `examples/monitoring/ingress-monitoring.yaml`
- `examples/code-server/ingress-code-server.yaml`

## Before you start

Decide these values once and reuse them across the runbook:

- Cluster hostname or IP
- Internal domain, for example `example.internal`
- Ingress class, for example `nginx`
- Storage class, for example `microk8s-hostpath`
- Whether you need `metallb`
- Whether you will use self-signed TLS, an internal CA, or no TLS
  during first bring-up

Create a simple environment worksheet:

```text
CLUSTER_NAME=microk8s
INGRESS_CLASS=nginx
BASE_DOMAIN=example.internal
ARGOCD_HOST=argocd.example.internal
GITEA_HOST=git.example.internal
PROMETHEUS_HOST=prometheus.example.internal
GRAFANA_HOST=grafana.example.internal
LINKERD_HOST=linkerd.example.internal
CODE_SERVER_HOST=code.example.internal
PERMISSION_MANAGER_HOST=permissions.example.internal
```

## Tested intent

This repo has been cleaned up for current Kubernetes APIs, but it is still not
a certified compatibility matrix. Treat these as required validation items:

- Record the actual MicroK8s snap channel you install
- Record the Kubernetes version from `kubectl version`
- Record the chart and app versions you install
- Keep the final chosen values files in Git

## Phase 1: Base cluster

Install MicroK8s:

```bash
sudo snap install microk8s --classic
sudo usermod -a -G microk8s "$USER"
newgrp microk8s
microk8s status --wait-ready
```

Export kubeconfig:

```bash
mkdir -p ~/.kube
microk8s config > ~/.kube/config
chmod 600 ~/.kube/config
```

Optional aliases:

```bash
alias kubectl='microk8s kubectl'
alias helm='microk8s helm3'
```

Enable the baseline addons:

```bash
microk8s enable dns helm3 ingress metrics-server hostpath-storage
```

Enable `metallb` only if you need `LoadBalancer` services:

```bash
microk8s enable metallb
```

Validation checkpoint:

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get storageclass
kubectl get pods -n ingress
kubectl get pods -n kube-system
```

Do not continue until the cluster is healthy.

## Phase 2: Access and naming

Before exposing any app:

- Create DNS or local host-file entries for the hostnames you picked
- Decide whether first access will be `port-forward` or ingress
- Decide how TLS secrets will be created

Rules for the rest of the runbook:

- Prefer `port-forward` for first login and admin bootstrap
- Prefer `spec.ingressClassName` over legacy ingress annotations
- Do not use Kubernetes Dashboard skip-login
- Do not rely on old `default-token` secret behavior

## Phase 3: Argo CD

Follow:

- [`argocd/README.md`](argocd/README.md)

Suggested first path:

1. Install Argo CD
2. Access it with `kubectl port-forward`
3. Confirm the admin secret works
4. Only then apply
   [`examples/argocd/ingress-argocd.yaml`](examples/argocd/ingress-argocd.yaml)

Validation checkpoint:

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
kubectl get ingress -n argocd
```

## Phase 4: Gitea

Follow:

- [`gitea/README.md`](gitea/README.md)

Suggested first path:

1. Copy and edit
   [`examples/gitea/gitea-values.yaml`](examples/gitea/gitea-values.yaml)
2. Install the Helm chart with that file
3. Confirm pods, PVCs, and services are healthy
4. Access by port-forward or the chart-managed ingress

Validation checkpoint:

```bash
kubectl get pods -n gitea
kubectl get pvc -n gitea
kubectl get svc -n gitea
kubectl get ingress -n gitea
```

## Phase 5: Monitoring

Follow:

- [`prometheus/README.md`](prometheus/README.md)

This repo does not yet pin one monitoring install method. For now, the runbook
assumes you already have Prometheus and Grafana running in `monitoring`.

Suggested first path:

1. Confirm the monitoring services and ports that actually exist
2. Update
   `examples/monitoring/ingress-monitoring.yaml`
3. Apply the ingress

Validation checkpoint:

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl get ingress -n monitoring
```

## Phase 6: Optional services

Linkerd:

- [`linkerd/README.md`](linkerd/README.md)
- Start with `linkerd check --pre`
- Only expose the dashboard after `linkerd check` and
  `linkerd viz check` both pass

Permission Manager:

- [`permission-manager/README.md`](permission-manager/README.md)
- Update [`permission-manager/values.yaml`](permission-manager/values.yaml)
- Render the chart before install with `helm template`

code-server:

- [`code-server/README.md`](code-server/README.md)
- Use only after you choose and pin a packaging path

## Recommended install order summary

```text
MicroK8s
-> DNS / ingress / storage / metrics
-> Argo CD
-> Gitea
-> Monitoring ingress
-> Linkerd (optional)
-> Permission Manager (optional)
-> code-server (optional)
```

## Definition of done

The stack is only "up" when all of the following are true:

- `kubectl get pods -A` shows no unexpected crash loops or pending workloads
- Each namespace has a known access path
- DNS or host-file entries resolve for every published hostname
- Ingress objects point to real services and correct ports
- Admin bootstrap credentials have been rotated or replaced where appropriate
- Final values files and manifests used for the install are stored in Git

## What is still left

- code-server is still a guided placeholder
- No values override files for Argo CD or Linkerd
- No compatibility matrix

Those are the next steps if you want this to become a reproducible stack, not
just a usable runbook.
