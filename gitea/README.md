Gitea is a community managed lightweight code hosting solution written in Go.
First we'll be creating a dedicated namespace in which we will be installing gitea
https://docs.gitea.io/en-us/install-on-kubernetes/

```
kubectl create namespace gitea
```

### Add gitea helm chart
Next we will be adding the helm charts
```
helm repo add gitea-charts https://dl.gitea.io/charts/
helm install gitea -n gitea gitea-charts/gitea
```
### Export the gitea helm values so we can update them.
```
helm show values gitea-charts/gitea > gitea-values.yaml
```

### Edit gitea chart values file
Edit the gitea-values.yaml file to turn on ingress ( example ingress section shown below )
```
ingress:
  enabled: true 
  annotations: 
    kubernetes.io/ingress.class: public 
    # kubernetes.io/tls-acme: "true"
  hosts:
    - git.brister.lan
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - git.example.com
```
### Apply the updated gitea-values.yaml file
```
helm upgrade -f values.yaml gitea -n gitea gitea-charts/gitea
```

### Check if gitea is running
```
kubectl get pods -n gitea
```

### Now that we have gitea running
Let's create an ingress controller so that we can access it.

First we need to get the name of the service we're adding to the ingress controller
```
kubectl get svc -n gitea
```

Create an ingress-gitea.yaml file 
```
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitea 
  namespace: gitea 
  annotations:
    # use the shared ingress-nginx
    kubernetes.io/ingress.class: public
spec:
  # https://kubernetes.io/docs/concepts/services-networking/ingress/
  # https://kubernetes.github.io/ingress-nginx/user-guide/tls/
  rules:
  - host: git.brister.lan
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gitea-http
            port:
              number: 80
```

Apply the ingress-gitea.yaml file
```
kubectl apply -f ingress-gitea.yaml
```
