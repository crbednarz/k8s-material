# Day 4 - Lab 2

With an understanding of operators, we can now try to work with one of the most popular options: cert-manager


## Steps

### Installing cert-manager

Similar to our last lab, we'll start with a helm install.

```sh
# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io --force-update

# Install the cert-manager helm chart
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.20.1 \
  --set crds.enabled=true
```

We can validate our install in a few ways:

```sh
# Check CRDs with "cert-manager" in the name
# We expect to see 6~ entries
kubectl api-resources | grep cert-manager

# We can also check cert-manager's pods:
kubectl get pods -n cert-manager
```

### Creating an issuer

Before we can get cert-manager to sign certificates, we need to tell it how. We do this through either an `Issuer` or `ClusterIssuer`, with the former being namespace scoped.

Here's the most basic ClusterIssuer:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
```

We can now put this into `issuer.yaml` and apply it:

```sh
kubectl apply -f issuer.yaml
```

Let's see if we can find it with kubectl:
```sh
kubectl get clusterissuers

# We should see something like:
# NAME                        READY   AGE
# selfsigned-cluster-issuer   True    26s
```


### Creating a Certificate

With our issuer in place, we now need to describe the certificate we'd like using a `Certificate` resource.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
spec:
  secretName: selfsigned-cert-tls
  issuerRef:
    name: selfsigned-cluster-issuer
    kind: ClusterIssuer
  commonName: nginx.local
  dnsNames:
    - nginx.local
```

We can save this file as `certificate.yaml` and deploy it with the following:

```sh
kubectl apply -f certificate.yaml
```

Let's validate this worked:
```sh
kubetl get certificates
# We expect to see one entry:
# NAME              READY   SECRET                AGE
# selfsigned-cert   True    selfsigned-cert-tls   3s

# We should also find a secret of type "kubernetes.io/tls" to go with it:
kubectl get secrets
```

### Attaching our certificate

In order to use our newly created certificate, we'll need to deploy a service that can consume it. Since we're already familiar with it, let's use Nginx.

First, we'll create a ConfigMap with the details Nginx needs.

Save the following as `nginx-config.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
        listen 443 ssl;
        server_name nginx.local;

        ssl_certificate /etc/nginx/certs/tls.crt;
        ssl_certificate_key /etc/nginx/certs/tls.key;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
```

Note the key lines here are these:
```
ssl_certificate /etc/nginx/certs/tls.crt;
ssl_certificate_key /etc/nginx/certs/tls.key;
```
This tells Nginx where to find `tls.crt` and `tls.key`. These are files we'd typically need to manually manage.

Now let's define our pod in `tls-nginx.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tls-nginx
  labels:
    app: tls-nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 443
    volumeMounts:
    - name: tls
      mountPath: /etc/nginx/certs
      readOnly: true
    - name: nginx-config
      mountPath: /etc/nginx/templates/default.conf.template
      subPath: nginx.conf
  volumes:
  - name: nginx-config
    configMap:
      name: nginx-config
  - name: tls
    secret:
      secretName: selfsigned-cert-tls
```


Notice how we pull in our cert files from a secret with these lines:
```yaml
  - name: tls
    secret:
      secretName: selfsigned-cert-tls
```

This secret can now be automatically updated for us by cert-manager.

Let's make sure our pod works:

```sh
kubectl apply -f tls-nginx.yaml

kubectl port-forward pod/tls-nginx 8443:443
```

Now browse to: https://localhost:8443/

You should receive a warning about an untrusted certificate. This is expected, as we're self-signing.

Digging into your browser's certificate details panel, we should find the common name as `nginx.local`, indicating our cert was successfully deployed.


### Cycling certificates

With the certificate deployed, we can play around with it a bit.

First we'll delete the secret itself:

```sh
kubectl delete secret selfsigned-cert-tls
```

If we wait a moment we'll find that our secret is magically back:
```sh
kubectl get secrets
```

This is because the underlying Certificate resource is still around. cert-manager, seeing a missing secret, will simply issue a new one.

Depending on your configuration, cert-manager may even do this for you to help cycle certificates near the end of their life.
