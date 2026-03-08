# Gitea on MicroK8s

This guide installs Gitea by Helm into its own namespace and uses the chart's
own ingress settings rather than layering a second manual ingress on top.

## Overview

Gitea is a lightweight self-hosted Git service. For a MicroK8s cluster, keep
the initial deployment simple: install the chart, set only the values you need,
then verify persistence, ingress, and login.

## Prerequisites

- A working MicroK8s cluster
- `kubectl` and `helm` pointed at that cluster
- Ingress enabled if you want browser access through a hostname
- Persistent storage available for Gitea data
- A hostname reserved for Gitea such as `git.example.internal`

## Install

Create the namespace:

```bash
kubectl create namespace gitea
```

Add the Helm repository and update repo metadata:

```bash
helm repo add gitea-charts https://dl.gitea.io/charts/
helm repo update
```

Export the default chart values so you can make targeted changes:

```bash
helm show values gitea-charts/gitea > gitea-values.yaml
```

Edit `gitea-values.yaml` for your environment. Start with the minimum changes
required for ingress and persistence.

Reference example:

- [`examples/gitea/gitea-values.yaml`](../examples/gitea/gitea-values.yaml)

Example values:

```yaml
ingress:
  enabled: true
  className: nginx
  annotations: {}
  hosts:
    - host: git.example.internal
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: gitea-tls
      hosts:
        - git.example.internal
```

Install the chart with your values file:

```bash
helm install gitea -n gitea gitea-charts/gitea -f gitea-values.yaml
```

If you are iterating on configuration after the first install, use:

```bash
helm upgrade gitea -n gitea gitea-charts/gitea -f gitea-values.yaml
```

## Expose

Use the chart-managed ingress rather than creating a second ingress manifest by
hand. That keeps the release configuration in one place.

If you do not want ingress yet, leave `ingress.enabled: false` and use
port-forward for first access:

```bash
kubectl port-forward svc/gitea-http -n gitea 3000:3000
```

Then open:

```text
http://127.0.0.1:3000
```

Notes:

- Replace `className: nginx` if your cluster uses another ingress class.
- Decide whether SSH access for Git operations should be exposed separately.
- Do not publish Gitea externally without TLS, authentication, and backup
  planning.

## Verify

Check the release resources:

```bash
kubectl get pods -n gitea
kubectl get svc -n gitea
kubectl get ingress -n gitea
helm list -n gitea
```

Inspect the generated values in the release if behavior does not match what you
expected:

```bash
helm get values gitea -n gitea
```

## Operate

Common day-two commands:

```bash
kubectl logs -n gitea deployment/gitea
kubectl describe pod -n gitea
helm status gitea -n gitea
```

Keep your chart version and app version pinned once you move past local
experimentation.

## Remove

Remove the release:

```bash
helm uninstall gitea -n gitea
```

Remove the namespace if you no longer need it:

```bash
kubectl delete namespace gitea
```

## Troubleshooting

- If the ingress exists but the site is unreachable, confirm the hostname
  resolves to your ingress controller and that the ingress class matches your
  cluster.
- If pods are pending, inspect PVC and storage class status before changing the
  application config.
- If upgrades fail, compare your `gitea-values.yaml` with the chart's current
  defaults instead of copying old examples forward unchanged.
