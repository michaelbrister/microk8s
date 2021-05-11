# Microk8s up and running on Ubuntu 20.04

### Install microk8s
sudo snap install microk8s --classic

### Enable microk8s add-ons
microk8s enable dashboard dns helm3 ingress metallb metrics-server prometheus registry storage

### Get the microk8s kubernetes dashboard token
```
token=$(microk8s.kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
microk8s.kubectl -n kube-system describe secret $token
```

### Alternatively
Edit kubernetes dashboard to have the - --enable-skip-login set so the token is not required

microk8s.kubectl edit deployment/kubernetes-dashboard --namespace=kube-system
and add enable-skip-login as below:
```
spec:
      containers:
      - args:
        - --auto-generate-certificates
        - --enable-skip-login
        image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
```

### Create kubernetes dashboard ingress
Now that we have the Kubernetes dashboard installed and we have the token ( or bypassed the need for the token ). Let's create a kubernetes ingress so we can access the dashboard.
Create a ingress-dashboard.yaml yaml file

```
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard
  namespace: kube-system
  annotations:
    # use the shared ingress-nginx
    kubernetes.io/ingress.class: public
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/configuration-snippet: |
      rewrite ^(/dashboard)$ $1/ redirect;
spec:
  # https://kubernetes.io/docs/concepts/services-networking/ingress/
  # https://kubernetes.github.io/ingress-nginx/user-guide/tls/
  rules:
  - http:
      paths:
      - path: /dashboard(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 443
```

### Apply the dashboard ingress
```
microk8s kubectl apply -f ingress-dashboard.yaml
```

### Access the kubernetes dashboard
You should now be able to access the dashboard by going to the microk8s server ip /dashboard
```
http://x.x.x.x/dashboard
```

Now that we have the dashboard working, let's install someone more useful.
We'll be using helm to install gitea. Gitea is a community managed lightweight code hosting solution written in Go.
First we'll be creating a dedicated namespace in which we will be installing gitea
https://docs.gitea.io/en-us/install-on-kubernetes/

```
microk8s kubectl create namespace gitea
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
microk8s kubectl get pods -n gitea
```

### Now that we have gitea running
Let's create an ingress controller so that we can access it.

First we need to get the name of the service we're adding to the ingress controller
```
microk8s kubectl get svc -n gitea
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
microk8s kubectl apply -f ingress-gitea.yaml
```

### Monitoring
Things are running, let's add ingress for prometheus ( monitoring is important )
Create an ingress file for prometheus
ingress-monitoring.yaml
```
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus 
  namespace: monitoring 
  annotations:
    # use the shared ingress-nginx
    kubernetes.io/ingress.class: public
spec:
  # https://kubernetes.io/docs/concepts/services-networking/ingress/
  # https://kubernetes.github.io/ingress-nginx/user-guide/tls/
  rules:
  - host: prometheus.brister.lan
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
  annotations:
    # use the shared ingress-nginx
    kubernetes.io/ingress.class: public
spec:
  # https://kubernetes.io/docs/concepts/services-networking/ingress/
  # https://kubernetes.github.io/ingress-nginx/user-guide/tls/
  rules:
  - host: grafana.brister.lan
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

Apply the ingress-monitoring.yaml file
```
microk8s kubectl apply -f ingress-monitoring.yaml
```
