# Day 5 - Lab 2

For this exercise, we'll test our a few security features:

- A simple network test of restricting traffic from a pod
- Scanning a container image
- Scanning our Kubernetes cluster


## Steps


### Prep

In order for network policies to work, we need to use a different CNI than minikube's default.

Thankfully, we can do that with an extra start flag:

```sh
minikube stop

minikube start --cni=calico
```

### Using a NetworkPolicy

First, let's test without a NetworkPolicy:

```sh
kubectl run test --rm -it --image busybox:latest -n day-5 -- sh -c "wget -qO - https://httpbin.org/get"
# We should see a valid response
```

Now let's make a NetworkPolicy that blocks all egress traffic. We'll write the following to `network-policy.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: day-5
spec:
  podSelector:
    matchLabels:
      project: policy-test
  policyTypes:
  - Egress
```

With the policy created, we can run our test:

```sh
# Run the same command as before
kubectl run test --rm -it --image busybox:latest -n day-5 -- sh -c "wget -qO - https://httpbin.org/get"
# As we don't match the policy label, this should still work

# Not only was our traffic to httpbin blocked, but so was our access to DNS! httpbin.org won't resolve.
kubectl run test --rm -it --image busybox:latest --labels "project=policy-test" -n day-5 -- sh -c "wget -qO - -T 5 https://httpbin.org/get"
# Expect to wait 10-30 seconds for a timeout to occur
```


### Scanning container images

Next up we'll perform a simple container image scan. There are many tools for this job, but Grype does a great job with terminal output.

Typically we'd install Grype (or Trivy) on our target system. To save time, we'll use the dockerized version of it.

(For this exercise we'll use the "nginx:latest" image to test against. Feel free to swap this out for the image of your choice)

```sh
# If you're curious about the "-v" flag, this shares access to the host's Docker
# server with the container.
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  anchore/grype \
    nginx:latest

# This will likely take a moment. "Real" environments would create a volume to hold
# Grype cache as well. Currently subsequent calls will trigger a re-download of the
# Grype vulnerability database.
```

### Scanning Kubernetes

As our last test, we'll check our Kubernetes configuration using kubescape.

To install kubescape, we'll use the command from the official instructions:
```sh
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash

# You'll need to follow the instructions after the install.
# It should give you a command to execute that looks like:
# export PATH=$PATH:/home/sysadmin/.kubescape/bin
```

Then we'll run it:

```sh
kubescape scan
```
