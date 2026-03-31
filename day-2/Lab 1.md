# Day 2 - Lab 1 (ConfigMap & Secrets)

Up to this point we've deployed the Nginx pod without any further configuration.
Let's see if we can add some functionality using `ConfigMap` and `Secret`.

We can break this into three parts:  

- **Pod** - This will hold our Nginx HTTP server
- **ConfigMap** - We'll store our basic configuration, a special message, and an HTML document here
- **Secret** - Here we'll store an htpasswd file, which triggers a username/password prompt


## Steps


### Prep

Before starting, be sure to make sure minikube is running:

```sh
minikube status

# if needed:
minikube start
```

### ConfigMap

First, let's start with out our ConfigMap. As stated above, we want to store three things in it:

1. A special message (`hello world`)
2. An nginx.conf file, which tells nginx how to behave. (`files/nginx.conf`)
3. An index.html file, which is the page we'll see when browsing to our server. (`files/index.html`)

Note: It is not typical to store html files within a ConfigMap, as websites are usually more than a single file.

While we could write our ConfigMap by hand, it's far more common to use the CLI to generate it for us:

```sh
# Note that the command depends on the path to nginx.conf and index.html
# If not executing from this directory, you'll need to ensure those files are present
kubectl create configmap nginx-config \
  --from-file nginx.conf=files/nginx.conf \
  --from-file index.html=files/index.html \
  --from-literal message="hello world" \
  --dry-run=client -o yaml > nginx-configmap.yaml
```

Take a moment to look at `nginx-configmap.yaml`. Can you find where each component is stored?

Let's go ahead and deploy this ConfigMap:

```sh
kubectl apply -f nginx-configmap.yaml
```


### Secret

Like ConfigMap, secrets are typically generated using the CLI. We can do that using the following command:

```sh
kubectl create secret generic nginx-secret \
  --from-file .htpasswd=files/htpasswd \
  --dry-run=client -o yaml > nginx-secret.yaml
```

If you look at `nginx-secret.yaml`, you'll notice that the htpasswd file contents have automatically been base64 encoded for us.

Let's deploy this one as well:

```sh
kubectl apply -f nginx-secret.yaml
```

### Pod

With our ConfigMap and Secret in place, we can finally get to our pod definition.

Let's take a look at a basic Pod definition:
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
```

First let's add our special message from the ConfigMap. We'll add it as an environment variable.

```diff
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
+     env:
+     - name: MESSAGE
+       valueFrom:
+         configMapKeyRef:
+           name: nginx-config
+           key: message
```


Next we'll make our pod aware of the ConfigMap and Secret so we can mount them:
```diff
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
+   volumes:
+   - name: nginx-config-volume
+     configMap:
+       name: nginx-config
+   - name: nginx-secret-volume
+     secret:
+       secretName: nginx-secret
```

Finally, we'll mount our files to the places they belong. You'll notice `subPath` here being used to specify which file we're pulling from the ConfigMap/Secret.


```diff
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
+     volumeMounts:
+     - name: nginx-config-volume
+       mountPath: /etc/nginx/templates/default.conf.template
+       subPath: nginx.conf
+     - name: nginx-config-volume
+       mountPath: /usr/share/nginx/html/index.html
+       subPath: index.html
+     - name: nginx-secret-volume
+       mountPath: /etc/nginx/.htpasswd
+       subPath: .htpasswd
    volumes:
    - name: nginx-config-volume
      configMap:
        name: nginx-config
    - name: nginx-secret-volume
      secret:
        secretName: nginx-secret
```

Here's the final pod YAML for reference:
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

Now let's run our pod (assuming we saved the above definition as `nginx-pod.yaml`)
```sh
kubectl apply -f nginx-pod.yaml
```

### Testing

With everything in place we should now be able to port-forward and test our configuration:

```sh
kubectl port-forward pod/nginx 8080:80
```

Now browse to http://localhost:8080/
You'll be prompted for a username and password, thanks to the .htpasswd file:  
- Username: user
- Password: password
