# Argo CD on MicroK8s

This guide installs Argo CD into a dedicated namespace and exposes it either by
temporary port-forwarding or by an ingress you manage. It is written as the
reference format for the other component guides in this repository.

## Overview

Argo CD provides GitOps-style application delivery for Kubernetes. In a
MicroK8s environment, keep the first install simple: install the control plane,
verify it is healthy, then decide how you want to expose it.

## Prerequisites

- A working MicroK8s cluster
- `kubectl` pointed at that cluster
- Ingress enabled if you plan to publish Argo CD through HTTP or HTTPS
- A hostname reserved for Argo CD such as `argocd.example.internal`
- A TLS plan if this will be reachable outside a trusted network

## Install

Create the namespace:

```bash
kubectl create namespace argocd
```

Apply the upstream install manifest:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for the control plane to come up:

```bash
kubectl get pods -n argocd
kubectl rollout status deployment/argocd-server -n argocd
kubectl rollout status deployment/argocd-repo-server -n argocd
kubectl rollout status deployment/argocd-application-controller -n argocd
```

## Expose

### Option A: local access with port-forward

This is the safest default for first-time setup:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open:

```text
https://127.0.0.1:8080
```

### Option B: ingress

If you want stable in-cluster or LAN access, create an ingress manifest and
replace the hostname and TLS details with values for your environment.

Reference example:

- [`examples/argocd/ingress-argocd.yaml`](/Users/mike/Documents/src/microk8s/examples/argocd/ingress-argocd.yaml)

Example `ingress-argocd.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - argocd.example.internal
      secretName: argocd-server-tls
  rules:
    - host: argocd.example.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  name: https
```

Apply it:

```bash
kubectl apply -f ingress-argocd.yaml
```

Notes:

- Change `ingressClassName` if your cluster does not use `nginx`.
- Do not assume a TLS secret already exists unless you created it or your
  ingress flow provisions it for you.
- Keep Argo CD behind authentication and TLS if it is reachable from anything
  broader than a private lab network.

## Verify

Confirm the namespace is healthy:

```bash
kubectl get all -n argocd
kubectl get ingress -n argocd
```

Retrieve the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Check the server service:

```bash
kubectl get svc argocd-server -n argocd
```

## Operate

Common day-two commands:

```bash
kubectl get applications -n argocd
kubectl logs deployment/argocd-server -n argocd
kubectl logs deployment/argocd-application-controller -n argocd
```

If you expose Argo CD by ingress, verify DNS resolution and certificate status
separately from Kubernetes object health.

## Remove

Remove the ingress if you created one:

```bash
kubectl delete -f ingress-argocd.yaml
```

Remove Argo CD:

```bash
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd
```

## Troubleshooting

- If the server never becomes ready, inspect events with `kubectl describe pod`
  in the `argocd` namespace.
- If ingress exists but the app is unreachable, confirm the hostname resolves
  to your ingress controller and that the ingress class matches your cluster.
- If login fails, regenerate or re-read the `argocd-initial-admin-secret`
  before changing anything else.
- If you plan to productionize this setup, move from the raw install manifest
  to a version-pinned workflow and keep your Argo CD configuration in Git.
