# Microk8s up and running on Ubuntu 20.04

### Install microk8s
sudo snap install microk8s --classic

### Create some command line alias
snap alias microk8s.kubectl kubectl
snap alias microk8s.helm3 helm
snap alias microk8s.istioctl istioctl

OR

alias kubectl='microk8s kubectl'
alias helm='microk8s helm3'

Update the .kube/config
sudo microk8s config > ~/.kube/config

### Enable microk8s add-ons
microk8s enable dashboard dns helm3 ingress metallb metrics-server prometheus registry storage

### Get the microk8s kubernetes dashboard token
```
token=$(microk8s.kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
microk8s.kubectl -n kube-system describe secret $token
```
```
kubectl describe secrets -n ni-system kubernetes-dashboard-token-c2chm
```

### Alternatively
Edit kubernetes dashboard to have the - --enable-skip-login set so the token is not required
```
kubectl edit deployment/kubernetes-dashboard --namespace=kube-system
```
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
kubectl apply -f ingress-dashboard.yaml
```

### Access the kubernetes dashboard
You should now be able to access the dashboard by going to the microk8s server ip /dashboard
```
http://x.x.x.x/dashboard
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
kubectl apply -f ingress-monitoring.yaml
```
