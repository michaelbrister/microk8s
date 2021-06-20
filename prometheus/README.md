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
