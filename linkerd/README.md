# Linkerd on MicroK8s

This guide installs Linkerd and the Viz extension, then exposes the dashboard
using either port-forwarding or a separately managed ingress.

## Overview

Linkerd is a service mesh for Kubernetes. The most reliable first install path
is to validate the cluster, install the control plane, install Viz, and verify
health before exposing any UI.

## Prerequisites

- A working MicroK8s cluster
- `kubectl` pointed at that cluster
- A Linkerd CLI version compatible with the control plane you plan to install
- Ingress enabled only if you want browser access without port-forwarding
- A hostname reserved for the dashboard such as `linkerd.example.internal`

## Install

Install the Linkerd CLI using the current upstream method for your platform,
then confirm it works:

```bash
linkerd version
```

Validate the cluster before installation:

```bash
linkerd check --pre
```

Install the control plane:

```bash
linkerd install | kubectl apply -f -
```

Verify the control plane:

```bash
linkerd check
```

Install the Viz extension:

```bash
linkerd viz install | kubectl apply -f -
```

Verify again after Viz is installed:

```bash
linkerd check
linkerd viz check
```

Optional extensions such as Jaeger should be added only after the base install
is healthy.

## Expose

### Option A: local access with port-forward

This is the preferred default:

```bash
linkerd viz dashboard
```

If you want raw access to the web service instead:

```bash
kubectl port-forward svc/web -n linkerd-viz 50750:8084
```

### Option B: ingress

If you need stable internal access, create an ingress manifest and provide your
own authentication and TLS. Do not use a hard-coded `admin/admin` secret.

Reference example:

- [`examples/linkerd/ingress-linkerd.yaml`](/Users/mike/Documents/src/microk8s/examples/linkerd/ingress-linkerd.yaml)

Example `ingress-linkerd.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: linkerd-viz
  namespace: linkerd-viz
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - linkerd.example.internal
      secretName: linkerd-viz-tls
  rules:
    - host: linkerd.example.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 8084
```

Apply it:

```bash
kubectl apply -f ingress-linkerd.yaml
```

Notes:

- The exact ingress annotations you need depend on your controller and how you
  handle auth headers and TLS termination.
- Keep dashboard exposure limited to trusted networks unless you have a clear
  identity and TLS story.

## Verify

Confirm the Linkerd namespaces are healthy:

```bash
kubectl get pods -n linkerd
kubectl get pods -n linkerd-viz
linkerd check
linkerd viz check
```

Inspect the dashboard service:

```bash
kubectl get svc web -n linkerd-viz
kubectl get ingress -n linkerd-viz
```

## Operate

Common day-two commands:

```bash
linkerd stat deployments -A
linkerd viz top deploy -A
kubectl logs -n linkerd deploy/linkerd-destination
```

Keep the CLI and control plane versions aligned when you upgrade.

## Remove

Remove the ingress if you created one:

```bash
kubectl delete -f ingress-linkerd.yaml
```

Remove Viz:

```bash
linkerd viz uninstall | kubectl delete -f -
```

Remove the control plane:

```bash
linkerd uninstall | kubectl delete -f -
```

## Troubleshooting

- If `linkerd check --pre` fails, fix those cluster issues before applying any
  manifests.
- If Viz installs but the dashboard is empty, verify that metrics components
  are healthy in `linkerd-viz`.
- If ingress access fails, test the dashboard with `linkerd viz dashboard`
  first so you can separate application health from ingress problems.
