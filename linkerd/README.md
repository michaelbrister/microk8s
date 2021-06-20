# Linkerd up and running on Ubuntu 20.04 with Microk8s
Many of these steps were taken directly from the official linkerd getting started guide
https://linkerd.io/2.10/getting-started/#step-2-validate-your-kubernetes-cluster

### Install the Linkerd CLI
```
curl -sL https://run.linkerd.io/install | sh
```
Once installed, verify the CLI is running correctly with:
```
linkerd version
```

Step 2: Validate your Kubernetes cluster
```
linkerd check --pre
```

Step 3: Install the control plane onto your cluster
```
linkerd install | kubectl apply -f -
```

Now let’s wait for the control plane to finish installing. Wait for the control plane to be ready (and verify your installation) by running:
```
linkerd check
```

Next, we’ll install some extensions. Extensions add non-critical but often useful functionality to Linkerd. For this guide, we need the viz extension, which will install Prometheus, dashboard, and metrics components onto the cluster:
```
linkerd viz install | kubectl apply -f - # on-cluster metrics stack
```
Optionally, at this point you can install other extensions. For example:
```
## optional
linkerd jaeger install | kubectl apply -f - # Jaeger collector and UI
```

Once you’ve installed the viz extension and any other extensions you’d like, we’ll validate everything again:
```
linkerd check
```

Step 4: Explore Linkerd
To do this we'll be exposing the dashboard via an ingress.
Create a ingress-linkerd.yaml yaml file
( Username / Password is admin/admin )
```
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: web-ingress-auth
  namespace: linkerd-viz
data:
  auth: YWRtaW46JGFwcjEkbjdDdTZnSGwkRTQ3b2dmN0NPOE5SWWpFakJPa1dNLgoK
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: linkerd-viz
  annotations:
    kubernetes.io/ingress.class: public
    nginx.ingress.kubernetes.io/upstream-vhost: $service_name.$namespace.svc.cluster.local:8084
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Origin "";
      proxy_hide_header l5d-remote-ip;
      proxy_hide_header l5d-server-id;
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: web-ingress-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'
spec:
  rules:
  - host: linkerd.brister.lan
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
Apply the dashboard ingress
```
kubectl apply -f ingress-linkerd.yaml
```
