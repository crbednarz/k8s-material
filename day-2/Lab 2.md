# Day 2 - Lab 2 (Service & Ingress)

Expanding on our earlier lab, we'll now look at to add a Service and Ingress.

## Steps

### Service

Our service is responsible for exposing the Nginx pod to the rest of the cluster, as well as our future Ingress resource.

Since our pod is already labeled as `app: nginx`, we can select on that:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
```

Assuming the above is saved as `nginx-service.yaml`, we can now deploy it:

```sh
kubectl apply -f nginx-service.yaml
```

If we want to ensure the service is working, we can deploy a quick temporary pod:

```sh
kubectl run --rm -it nginx-test --image=busybox -- sh -c "sleep 2 && wget -qO - http://user:password@nginx.default.svc.cluster.local"
```

### Ingress


Before we can write the ingress spec, we'll need to enable an ingress controller in minikube:

```sh
minikube addon enable ingress
```

With the service complete, and minikube configured, we can now add an Ingress.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
spec:
  ingressClassName: nginx
  rules:
  - host: nginx.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```

If we save this as `nginx-ingress.yaml` we can then deploy it:

```sh
kubectl apply -f nginx-ingress.yaml
```

### Testing

In one terminal tell minikube to begin tunneling:

```sh
minikube tunnel
```

In another terminal, run the following commands to test if the ingress was correctly deployed:
```sh
curl --header 'Host: nginx.local' http://user:password@$(minikube ip)
```
