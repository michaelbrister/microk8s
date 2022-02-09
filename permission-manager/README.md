# Kubernetes Permission Manager
Kubernetes uses certificates to manager user permissions. 

To simplify this we'll use permission manager to create our users and assign them permissions.
https://github.com/sighupio/permission-manager/blob/master/docs/installation.md

## Create a namespace for permissions-manager
```
kubectl create namespace permission-manager
```

## Create kubernetes secret
```
---
apiVersion: v1
kind: Secret
metadata:
  name: permission-manager
  namespace: permission-manager
type: Opaque
stringData:
  PORT: "4000" # port where server is exposed
  CLUSTER_NAME: "my-cluster" # name of the cluster to use in the generated kubeconfig file
  CONTROL_PLANE_ADDRESS: "https://172.17.0.3:6443" # full address of the control plane to use in the generated kubeconfig file
  BASIC_AUTH_PASSWORD: "changeMe" # password used by basic auth (username is `admin`)
```

## Apply the files to install permission-manager
```
kubectl apply -f https://github.com/sighupio/permission-manager/releases/download/v1.7.1-rc1/crd.yml
kubectl apply -f https://github.com/sighupio/permission-manager/releases/download/v1.7.1-rc1/seed.yml
kubectl apply -f https://github.com/sighupio/permission-manager/releases/download/v1.7.1-rc1/deploy.yml
```

## Create an ingress for easy access ( optional - can utilize port-forward instead )
```
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: permission-manager
  namespace: permission-manager
  annotations:
    # use the shared ingress-nginx
    kubernetes.io/ingress.class: public
spec:
  # https://kubernetes.io/docs/concepts/services-networking/ingress/
  # https://kubernetes.github.io/ingress-nginx/user-guide/tls/
  rules:
  - host: permissions.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: permission-manager
            port:
              number: 4000
```
