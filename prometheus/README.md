# Prometheus and Grafana on MicroK8s

This guide focuses on exposing Prometheus and Grafana that already exist in a
`monitoring` namespace. It does not claim that MicroK8s itself installed those
services for you; verify the stack you are running before applying ingress.

## Overview

Monitoring setups vary more than the older notes in this repository assumed.
Before exposing any UI, confirm which chart or addon created the services and
what their actual service names are.

## Prerequisites

- A working MicroK8s cluster
- A monitoring stack already installed in the `monitoring` namespace
- `kubectl` pointed at that cluster
- Ingress enabled if you want browser access by hostname
- Hostnames reserved for Prometheus and Grafana such as
  `prometheus.example.internal` and `grafana.example.internal`

## Install

If you have not installed a monitoring stack yet, stop here and pick a specific
deployment path first. Typical options are:

- A MicroK8s addon, if the version you run still provides and supports it
- `kube-prometheus-stack` by Helm
- Another Prometheus operator or standalone chart you have already standardized

Once the stack exists, inspect the services before writing ingress manifests:

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

## Expose

Create an ingress manifest only after confirming the service names and ports in
your cluster.

Reference example:

- [`examples/monitoring/ingress-monitoring.yaml`](/Users/mike/Documents/src/microk8s/examples/monitoring/ingress-monitoring.yaml)

Example `ingress-monitoring.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus
  namespace: monitoring
spec:
  ingressClassName: nginx
  rules:
    - host: prometheus.example.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-k8s
                port:
                  number: 9090
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
spec:
  ingressClassName: nginx
  rules:
    - host: grafana.example.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana
                port:
                  number: 3000
```

Apply it:

```bash
kubectl apply -f ingress-monitoring.yaml
```

Notes:

- Replace service names if your monitoring stack uses different names.
- Add TLS and authentication before exposing these UIs beyond a trusted
  internal network.
- If your Grafana service listens on a different port, use the actual service
  definition from your cluster rather than the example above.

## Verify

Confirm the services and ingress objects:

```bash
kubectl get svc -n monitoring
kubectl get ingress -n monitoring
kubectl get pods -n monitoring
```

If you need to verify locally before DNS is ready, port-forward either service:

```bash
kubectl port-forward svc/grafana -n monitoring 3000:3000
kubectl port-forward svc/prometheus-k8s -n monitoring 9090:9090
```

## Operate

Common day-two commands:

```bash
kubectl logs -n monitoring deploy/grafana
kubectl get endpoints -n monitoring
kubectl describe ingress -n monitoring
```

Treat dashboards and alerting rules as application configuration and keep them
versioned alongside the monitoring stack when possible.

## Remove

Remove the ingress objects:

```bash
kubectl delete -f ingress-monitoring.yaml
```

Do not remove the monitoring namespace unless you also intend to remove the
monitoring stack itself.

## Troubleshooting

- If the ingress exists but traffic fails, confirm the backing service names and
  ports match what is actually running in the cluster.
- If Grafana loads but dashboards are missing, that is usually a stack or data
  source issue, not an ingress issue.
- If Prometheus is unreachable, check whether the service is `ClusterIP` only
  and whether the backing pods are healthy before editing ingress.
