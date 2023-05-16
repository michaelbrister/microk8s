Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes.

### Create the ArgoCD namespace
```
kubectl create namespace argocd
```

### Install ArgoCD
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Create an ingress-argocd.yaml file 
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-http-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: public
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
    host: argocd.brister.lan
  tls:
  - hosts:
    - argocd.brister.lan
    secretName: argocd-secret # do not change, this is provided by Argo CD
```

Apply the ingress-argocd.yaml file
```
kubectl apply -f ingress-argocd.yaml
```
### Get the default password for ArgoCD
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```
