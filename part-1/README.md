# Part 1


Use the following to create an Nginx container:
```sh
kubectl run nginx --image=nginx
```

Test out some other commands:
```sh
# Get more details about nginx
kubectl get pods -o wide

# Even more details!
kubectl describe pod nginx
# Can you find where the image is listed?

# Get pods in all namespaces
kubectl get -A pods

# Get all namespaces
kubectl get ns

# Get common resource types in the system namespace
kubectl get all -n kube-system

# What happens when we try to edit the pod?
kubectl edit pod nginx
```

Clean up after:
```sh
kubectl delete pod nginx
```


Bonus points:
```sh
# Create a new namespace
kubectl create ns mynamespace

# Create nginx under the new namespace
kubectl run nginx --image=nginx -n mynamespace

# Delete the namespace
kubectl delete ns mynamespace

# Is the pod gone too?
```
