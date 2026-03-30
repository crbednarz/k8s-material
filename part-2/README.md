# Part 2

Generate a YAML file for an Nginx pod using a dry-run command:
```sh
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

Be sure to take a look inside `pod.yaml`, then deploy it with:
```sh
kubectl apply -f ./pod.yaml
```

To access it from the browser, use `kubectl port-forward`:
```sh
kubectl port-forward pod/nginx 8080:80
# We can now browser to localhost:8080
```

1. Try adding a label by placing the following snippet under `metadata`:

```yaml
labels:
  app: nginx-lab
  owner: your_name
```

Can you find it with the following?
```sh
kubectl get pods -l app=nginx-lab
```

2. Try adding this livenessProbe snippet in the container definition:

```yaml
livenessProbe:
  httpGet:
    path: /index.html
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

Can you find the ready state with the following?
```sh
kubectl describe pod nginx
```

Can you update the livenessProbe to always fail? What does `kubectl describe` report now?

3. Try adding a port definition in the container definition: (This won't impact port-forward)

```yaml
ports:
- containerPort: 80
```

Can you find where the port is listed with `kubectl describe pod nginx`?

Bonus points:

Can you figure out how to use `kubectl logs` to get the logs from our pod?
