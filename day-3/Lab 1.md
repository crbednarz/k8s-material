# Day 3 - Lab 1

With our recent understanding of the components that make up Kubernetes, we'll now see if we can identify some of these components on our minikube deployment.


## Steps


### Prep

Before starting, be sure to make sure minikube is running:

```sh
minikube status

# if needed:
minikube start
```

### Control Plane

The control plane services exist as pods within the `kube-*` namespaces.

We can find many of the components with the following command:
```sh
kubectl get pods -n kube-system
```

You should see:

- API Server (kube-apiserver-minikube)
- Controller Manager (kube-controller-manager-minikube)
- Scheduler (kube-scheduler-minikube)
- Etcd (kube-etcd-minikube)

Try gettings logs on one:

```sh
kubectl logs -n kube-system kube-apiserver-minikube
```

We can also make a request against the API server directly:
```sh
# Note: Use Ctrl+C to exit this once you're done with the link below
kubectl proxy --port=8001
```

Then browse to http://localhost/api/v1/namespaces/kube-system/pods


### Kubelet

Minikube operates a bit differently than a standard deployment, so we'll need to take an extra step to see what's going on:

```sh
# This will enter into the minikube environment
minikube ssh
```

From here we can find our Kubelet process:
```sh
# "ps aux" lists all running processes
# "| grep kubelet" will only display lines containing "kubelet"
ps aux | grep kubelet
```

Finally, we'll exit our minikube session:
```sh
exit
```
