# Permission Manager on MicroK8s

This guide documents the local Helm chart in this repository. The chart has
been cleaned up for current Kubernetes APIs, but it still needs real cluster
validation before it should be treated as production-ready.

## Overview

Permission Manager can simplify Kubernetes user and RBAC management. The local
chart now uses current ingress and autoscaling APIs and removes
PodSecurityPolicy-era RBAC references, but it still needs
environment-specific validation.

## Prerequisites

- A working MicroK8s cluster
- `kubectl` and `helm` pointed at that cluster
- A clear requirement for managing users through this tool rather than native
  RBAC manifests or another access workflow
- Willingness to validate the local chart in your own environment before using
  it broadly

## Install

Create the namespace:

```bash
kubectl create namespace permission-manager
```

Review and edit the chart values in
[`permission-manager/values.yaml`](values.yaml)
before installing. At minimum, replace placeholder values for:

- `config.clusterName`
- `config.controlPlaneAddress`
- `config.basicAuthPassword`
- ingress hostname and TLS settings

Install the local chart from the repository root:

```bash
helm install permission-manager -n permission-manager \
  -f permission-manager/values.yaml ./permission-manager
```

If you make further changes to the values file:

```bash
helm upgrade permission-manager -n permission-manager \
  -f permission-manager/values.yaml ./permission-manager
```

## Expose

The chart includes an ingress template. Before enabling external access, verify
the ingress values in
[`permission-manager/values.yaml`](values.yaml)
match your cluster.

You should update the ingress settings to use:

- A real hostname such as `permissions.example.internal`
- The correct ingress class for your controller
- TLS if the service is reachable beyond a tightly controlled internal network

## Verify

Check the release resources:

```bash
kubectl get pods -n permission-manager
kubectl get svc -n permission-manager
kubectl get ingress -n permission-manager
helm status permission-manager -n permission-manager
```

Inspect the rendered manifests before trusting the install on a modern cluster:

```bash
helm template permission-manager ./permission-manager \
  -f permission-manager/values.yaml
```

## Operate

Use this chart carefully. Improvements made in this repo include:

- Current `autoscaling/v2` HPA manifests
- `ingressClassName` support in the ingress template
- Removal of PodSecurityPolicy-era RBAC references
- Support for the corrected `config.controlPlaneAddress` value name

Placeholder values still must not be left unchanged, and the chart should still
be rendered and tested against the Kubernetes version you plan to run.

## Remove

Remove the release:

```bash
helm uninstall permission-manager -n permission-manager
```

Remove the namespace if you no longer need it:

```bash
kubectl delete namespace permission-manager
```

## Troubleshooting

- If the chart fails to install, inspect the rendered manifests first and
  confirm your cluster version and ingress settings match the chart output.
- If ingress comes up but the UI is unreachable, verify the configured ingress
  class, hostname resolution, and TLS secret handling.
- If authentication or cluster access fails, review the generated secret values
  and confirm the control plane address is correct for your environment.
