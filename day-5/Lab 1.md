# Day 5 - Lab 1

ServiceAccounts and RBAC are typically used for applications within a cluster to interact with the control plane.
To help make sense of this, we'll walk through the process of creating our own service account and using an interactive pod to test its permissions.

## Steps

### Prep

To begin with, make sure minikube is running:

```sh
minikube status

# if needed:
minikube start
```

### Testing the default ServiceAccount

By default, each namespace includes a service account which is automatically attached to every pod.

We can find this using `kubectl`

```sh
# Create a namespace for today
kubectl create namespace day-5

# You can also write "sa" instead of "serviceaccounts"
kubectl get serviceaccounts -n day-5

# We should see something like:
# NAME      SECRETS   AGE
# default   0         33s

kubectl describe sa default -n day-5
# Again, the results should be fairly empty:
# Name:                default
# Namespace:           day-5
# Labels:              <none>
# Annotations:         <none>
# Image pull secrets:  <none>
# Mountable secrets:   <none>
# Tokens:              <none>
# Events:              <none>
```

Since it's mounted by default, all we need to do is create a pod:

```sh
# This may take a moment as your machine downloads the busybox image
kubectl run --rm -it -n day-5 test --image busybox:latest -- /bin/sh

# Note: This puts us into an interactive shell inside the pod.
# Typing "exit" will close the connection and delete the pod. (due to "--rm")
```

Let's check out the default token:
```sh
cd /var/run/secrets/kubernetes.io/serviceaccount/

ls
# ca.crt     namespace  token

cat token
# We should see a bunch of random numbers and letters dump to the terminal
```

Since we've found our token, let's use it query the control plane

```sh
# First, we'll try without using the token:
wget -qO - https://kubernetes.default.svc/apis
# Connecting to kubernetes.default.svc (10.43.0.1:443)
# wget: note: TLS certificate validation not implemented
# wget: server returned error: HTTP/1.1 401 Unauthorized

# Next, let's load our token into a variable
TOKEN=$(cat token)

# And try with passing the appropriate HTTP header
wget -qO - --header "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/apis
# This should produce a bunch of JSON output
```

Unfortunately, the default permissions will fail if we try to do something like listing pods:
```sh
wget -qO - --header "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces/day-5/pods
# 403 Forbidden error
```

Let's go ahead and exit for now...
```sh
exit
```

### Creating a ServiceAccount and Role

As the default service account doesn't have the permissions we want, we'll make our own:

```sh
kubectl create serviceaccount my-account -n day-5
```

Of course, it doesn't have any permissions yet, so we'll need to create a Role for it.

In `role.yaml` we can define enough permissions to read pods in our namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: day-5
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

Now we'll apply and verify it:

```sh
kubectl apply -f role.yaml
# role.rbac.authorization.k8s.io/pod-reader created

kubectl get role -n day-5
# NAME         CREATED AT
# pod-reader   2026-04-13T20:56:26Z
```

Finally, we'll bind our new "pod-reader" role to "my-account" by creating a RoleBinding in `role-binding.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: day-5
subjects:
- kind: ServiceAccount
  name: my-account
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

We can verify it as well:

```sh
kubectl describe rolebinding read-pods -n day-5
# Name:         read-pods
# Labels:       <none>
# Annotations:  <none>
# Role:
#   Kind:  Role
#   Name:  pod-reader
# Subjects:
#   Kind            Name        Namespace
#   ----            ----        ---------
#   ServiceAccount  my-account
```


### Testing

With our account permissions configured, let's finally see if we can query pod details.

```sh
# We'll have to use "--overrides" to specify the service account for our interactive pod
kubectl run --rm -it --overrides='{ "spec": { "serviceAccountName": "my-account" } }' -n day-5 test --image busybox:latest -- /bin/sh

# Now let's get the token again
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

# And run our pod query command again:
wget -qO - --header "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces/day-5/pods
```

We should finally see a JSON dump from our query. The service account worked!
