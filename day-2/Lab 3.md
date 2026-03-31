# Day 2 - Lab 3 (Deployment)

As our final enhancement, let's make our nginx pod run as a deployment

## Steps

### Deployment

First, let's delete our current pod, as to not cause conflicts:

```sh
kubectl delete pod nginx
```

Now, let's take a look at the YAML for the pod we just deleted:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    env:
    - name: MESSAGE
      valueFrom:
        configMapKeyRef:
          name: nginx-config
          key: message
    volumeMounts:
    - name: nginx-config-volume
      mountPath: /etc/nginx/templates/default.conf.template
      subPath: nginx.conf
    - name: nginx-config-volume
      mountPath: /usr/share/nginx/html/index.html
      subPath: index.html
    - name: nginx-secret-volume
      mountPath: /etc/nginx/.htpasswd
      subPath: .htpasswd
  volumes:
  - name: nginx-config-volume
    configMap:
      name: nginx-config
  - name: nginx-secret-volume
    secret:
      secretName: nginx-secret
```

To make this a Deployment, we'll need to embed it into the template of our Deployment resource:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    # <Pod definition here>
```


Here they are combined:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
        env:
        - name: MESSAGE
          valueFrom:
            configMapKeyRef:
              name: nginx-config
              key: message
        volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/templates/default.conf.template
          subPath: nginx.conf
        - name: nginx-config-volume
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
        - name: nginx-secret-volume
          mountPath: /etc/nginx/.htpasswd
          subPath: .htpasswd
      volumes:
      - name: nginx-config-volume
        configMap:
          name: nginx-config
      - name: nginx-secret-volume
        secret:
          secretName: nginx-secret
```

This can then be saved as `nginx-deployment.yaml` and applied:

```sh
kubectl apply -f nginx-deployment.yaml
```

### Testing

With the pod replaced, we can now explore a bit. Here are some commands to try out:

```sh
kubectl scale deployment/nginx --replicas=3

kubectl rollout history deployment nginx
```

Will our previous ingress still work?

```sh
# Be sure we're running tunnel
minikube tunnel

# This should be routed to one of our pods
curl --header 'Host: nginx.local' http://user:password@$(minikube ip)
```

If we check the logs for each pod, do we see our requests distributed?


