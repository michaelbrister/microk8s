# code-server on MicroK8s

This guide is a starting point for running code-server in Kubernetes. The
previous file was only a list of links; this version makes the intended
deployment shape explicit without locking the repo to one unverified chart.

## Overview

code-server provides a browser-accessible VS Code environment. In Kubernetes,
the important choices are persistence, authentication, and how broadly the
service should be reachable.

## Prerequisites

- A working MicroK8s cluster
- `kubectl` and `helm` pointed at that cluster
- Ingress enabled if you want browser access through a hostname
- Persistent storage for user data and extensions
- A hostname reserved for code-server such as `code.example.internal`

## Install

Pick one maintained packaging path and stay consistent. Common options include:

- A Helm chart you have evaluated and pinned
- A direct deployment manifest you maintain in Git
- A container image workflow you package yourself

Before installing, decide:

- Where user data should be stored
- How login will be handled
- Whether the pod needs access to a Git SSH key, tokens, or other secrets

If you use Helm, export the default values from the chart you chose and keep
your edited values in a dedicated file such as `code-server-values.yaml`.

## Expose

For first access, prefer port-forward:

```bash
kubectl port-forward svc/code-server -n code-server 8080:8080
```

If you publish through ingress, use a hostname and TLS.

Reference example:

- [`examples/code-server/ingress-code-server.yaml`](/Users/mike/Documents/src/microk8s/examples/code-server/ingress-code-server.yaml)

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: code-server
  namespace: code-server
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - code.example.internal
      secretName: code-server-tls
  rules:
    - host: code.example.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: code-server
                port:
                  number: 8080
```

## Verify

Confirm the workload and service:

```bash
kubectl get pods -n code-server
kubectl get svc -n code-server
kubectl get pvc -n code-server
kubectl get ingress -n code-server
```

## Operate

- Keep the image version pinned.
- Back up the persistent volume if the environment matters.
- Keep secrets out of the image and inject them through Kubernetes secrets or
  your secret management workflow.

## Remove

Remove the release or manifests you installed, then remove the namespace only
after deciding whether the persistent volume should be retained.

## Troubleshooting

- If the editor starts but state disappears after restart, inspect PVC binding
  and mount paths first.
- If login works locally but not behind ingress, check TLS and proxy header
  behavior at the ingress controller.
- If extensions fail to install, confirm outbound network policy and container
  permissions.

## References

- [code-server upstream](https://github.com/coder/code-server)
- [linuxserver/code-server image](https://hub.docker.com/r/linuxserver/code-server)
- [Artifact Hub search for code-server charts](https://artifacthub.io/packages/search?ts_query_web=code-server)
