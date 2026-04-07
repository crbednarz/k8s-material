# Day 4 - Lab 1

After learning about Helm charts, it's time to try one out.

For this lab, we'll be using stefanprodan's podinfo chart.
This is a simple application with some easy to understand values to override.

## Stage 1 - Installing & Testing

### Prep

As always, before starting, be sure to make sure minikube is running:

```sh
minikube status

# if needed:
minikube start
```

### Helm chart repo

You may recall that HTTP and OCI are the two main protocols for installing Helm charts.
Though this project does include the simpler OCI chart, we'll be using HTTP, as it's good practice.

```sh
# Register repo as "stefanprodan"
helm repo add stefanprodan https://stefanprodan.github.io/podinfo

# Refresh our knowledge of registered repos
helm repo update
```

Now let's verify we've adding our repo correctly:
```sh
helm repo ls
# We expect to see "stefanprodan" listed
```

We can also double check what charts are under our repo:
```sh
helm search repo stefanprodan
# There should be one chart listed called "stefanprodan/podinfo"
```

Now let's see if we can find what version are available:
```sh
helm search repo -l stefanprodan/podinfo
# We should expect to see a long list of chart versions
```

### Installing the chart

With our repo set up, we can now install the chart.

To follow best practices, we'll create a `lab` namespace and install our chart into it.

Below are four different options to install our chart. Can you tell the differences?
```sh
# Option 1
helm install podinfo stefanprodan/podinfo -n lab --create-namespace

# Option 2
kubectl create namespace lab
helm install podinfo stefanprodan/podinfo -n lab

# Option 3
helm upgrade podinfo --install stefanprodan/podinfo -n lab --create-namespace


# Option 4
kubectl create namespace lab
helm upgrade podinfo --install stefanprodan/podinfo -n lab
```

Option 2 & 4 both invoke `kubectl create namespace lab`, which will fail if the namespace already exists.  
Options 1 & 2 both directly call `helm install`, which will fail if the chart has already been installed.

Remember, there are no "right" options here. While `helm upgrade --install` and `--create-namespace` fail less, you may want failures if something already exists.


### Testing install

With our chart installed using its default values, we can now test it out to see what to expect:

```sh
kubectl -n lab port-forward deploy/podinfo 8080:9898
# Exit this after you're done testing the page by pressing Ctrl+C
```

Now visit: http://localhost:8080/

## Stage 2 - Changing Values

### Getting values

While we'll learn about other methods of finding the values list later one, for now we can use the helm CLI.

```sh
# We'll use "tee" here to write the output to a file at the same time
helm show values stefanprodan/podinfo | tee all-values.yaml
```

Take a moment to look at the newly created `all-values.yaml` in your editor of choice.

You'll notice this block near the top:
```yaml
ui:
  color: "#34577c"
  message: ""
  logo: ""
```

This will act as a good jumping off point for changes.

### Changing individual values

Let's start by changing the color to a nice bright pink value of `#FF69B4`. Since our chart is already installed, we'll need to upgrade it:
```sh
# Note that ui.color refers to the "color" property under the "ui" block
helm upgrade podinfo stefanprodan/podinfo -n lab --set ui.color="#FF69B4"

# Redo our port-forward
kubectl -n lab port-forward deploy/podinfo 8080:9898
```

You may have noticed a `REVISION: 2` in the output. Remember that for later.

Now let's check the site again: http://localhost:8080/

If everything worked, it should be bright pink now.

Now let's change the message:

```sh
helm upgrade podinfo stefanprodan/podinfo -n lab --set ui.message="Hello World"

# Redo our port-forward
kubectl -n lab port-forward deploy/podinfo 8080:9898
```

We can now check the site once more: http://localhost:8080/

Our message is there now, but our color got reset! To have both at the same time we need one of these two options:

```sh
# Option 1
helm upgrade podinfo stefanprodan/podinfo -n lab --set ui.color="#FF69B4"
# "--reuse-values" ensures we keep the values we used in our last deploy
helm upgrade podinfo stefanprodan/podinfo -n lab --reuse-values --set ui.message="Hello World"

# Option 2
helm upgrade podinfo stefanprodan/podinfo -n lab --set ui.color="#FF69B4" --set ui.message="Hello World"


# Redo our port-forward
kubectl -n lab port-forward deploy/podinfo 8080:9898
```

Let's check again: http://localhost:8080/

## Stage 3 - Override Files

### Defining a values file

As we saw at the end of the last stage, managing values through the command line can be tricky- especially as we want to override more properties.

Why don't we use a values file instead.

Create a file called `podinfo-values.yaml` in your current directory. Inside this file we'll put the following:
```yaml
ui:
  message: Hello Kubernetes!
  color: "#BCA4E3" # How about purple this time?
```

Now we can upgrade our chart with our values file:
```sh
helm upgrade podinfo stefanprodan/podinfo -n lab -f podinfo-values.yaml

# Redo our port-forward
kubectl -n lab port-forward deploy/podinfo 8080:9898
```

Great, both values are in! We can double check: http://localhost:8080/

### Multiple values files

Now a single values file is great, but sometimes you may find yourself wanting more than one. Perhaps one overrides configuration further for a specific environment.

Let's make a second values YAML called `podinfo-dev-values.yaml` with the following:

```yaml
ui:
  color: "#F9E076"
  logo: "https://placecats.com/500/500"
  # We'll leave message alone
```

Now let's chain them together:
```sh
helm upgrade podinfo stefanprodan/podinfo -n lab -f podinfo-values.yaml -f podinfo-dev-values.yaml

# Redo our port-forward
kubectl -n lab port-forward deploy/podinfo 8080:9898
```

What if we swap the order?
```sh
helm upgrade podinfo stefanprodan/podinfo -n lab -f podinfo-dev-values.yaml -f podinfo-values.yaml

# Redo our port-forward
kubectl -n lab port-forward deploy/podinfo 8080:9898
```

## Stage 4 - Dry-run

### Getting Helm's output

Now that we've modified our values a bit, let's see if we can understand what our changes are doing.

First, we'll use `--dry-run` to find the "default" output of the helm install:
```sh
# Notice how "install" is okay here, as we aren't actually installing anything
helm install podinfo stefanprodan/podinfo -n lab --dry-run | tee base-install.yaml
```

Now let's do it again with our overrides:
```sh
helm install podinfo stefanprodan/podinfo -n lab -f podinfo-values.yaml -f podinfo-dev-values.yaml --dry-run | tee modified-install.yaml
```

With both outputs, let's now perform a diff between them:
```sh
git diff --no-index base-install.yaml modified-install.yaml

# Use the arrow keys to scroll up and down
# Press "q" to leave
```

You should be able to spot where our overrides have taken effect.
