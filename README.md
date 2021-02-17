# Hands-On

Deploy the sample Golang application [my-app](https://github.com/robinhuiser/my-app) in Kubernetes using GitOps principles.

> Based upon [Tutorial: Everything You Need To Become a GitOps Ninja - Alex Collins & Alexander Matyushentsev](https://www.youtube.com/watch?v=r50tRQjisxw), modified for the video series "Bank of Pi" to explain the concept of GitOps.

## Setup

1. Install the following utils:
   
   ~~~bash
   # Install Kustomize using brew https://brew.sh/
   $ brew install kustomize

   # Check the version (> 4.x)
   $ kustomize version
   ~~~

2. Fork the example repo

   * Open https://github.com/robinhuiser/my-app-deployment
   * Click “Fork”. 

3. Clone your fork

   ~~~bash
   # Set your username in an environment variable
   # This variable is later used to define Kubernetes resources
   export username=... ;# your Github username in lowercase

   # Clone and change into the directory
   $ git clone git@github.com:${username}/my-app-deployment.git
   $ cd my-app-deployment
   ~~~

> Assure you have [published you SSH public key](https://docs.github.com/en/github/authenticating-to-github/adding-a-new-ssh-key-to-your-github-account) for your GitHub account

## Initial build

First, we verify if our base configuration is working before creating any environment specific settings in next steps.

~~~bash
$ kustomize build base

apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - image: registry.tekqube.lan:32000/gitops-workshop/my-app:v1
    imagePullPolicy: IfNotPresent
    name: main
~~~

Next, we'll create an overlay directory:

~~~bash
$ mkdir -p overlays/dev
~~~

... and create the file `kustomization.yaml` under this directory to define changes we are about to apply compared to the base configuration:

~~~bash
# Create the initial version of our kustomization definition
$ cat > overlays/dev/kustomization.yaml <<EOL
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
EOL
~~~

Let's see if the build is happy and push to git:

~~~bash
# Test
$ kustomize build overlays/dev

apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - image: registry.tekqube.lan:32000/gitops-workshop/my-app:v1
    imagePullPolicy: IfNotPresent
    name: main

# Push to git
$ git add . && git commit -am "add dev overlay"
$ git push
~~~

> As you could see, the overlay outputs (for now) the same as the base configuration - this is just the starting point for modifications in the next chapters.

## Lab 1: Set a name prefix

In order to prevent clashes between resources in a namespace (we do not want someone else to delete or modify our resources), we can prefix our resource definitions.

Kustomize CLI has a feature which enables this in our `kustomization.yaml` file; as a result all our resources are automatically prefixed when we generate our manifests.

~~~bash
# Change to the directory containing your 
# customization.yaml file and add prefix
$ cd overlays/dev
$ kustomize edit set nameprefix ${username}-

# What was the result of this command?
$ git diff

# Run (and see updated output)
$ kustomize build

# Pushto Git
$ git commit -am "set name prefix"
$ git push
~~~

## Lab 2: Open Argo CD and create your app

* Open https://argocd.console.tekqube.lan/
* Login using admin
* Click "Create application"

| Field | Value |
|-------|-------|
| Application name: | `${username}` |
| Project: | `default` |
| Sync policy: | `Manual` |
| Repository: | `https://github.com/${username}/my-app-deployment` |
| Revision: | `HEAD` |
| Path: | `overlays/dev` |
| Cluster: | `https://kubernetes.default.svc` |
| Namespace: | `default` |
  
Next, sync your app by either:

* Click "Sync".
* Click "Synchronize" in the Sliding panel.

## Lab 2: Upgrade your app

For this exercise, we are going to set the image to something non-existent...

~~~bash
# Set the image
$ kustomize edit set image registry.tekqube.lan:32000/gitops-workshop/my-app:v2

# See the differences & build to verify changes
$ git diff
$ kustomize build
~~~

Next, we push this change to our Git repo:

~~~bash
# Commit & push
$ git commit -am "upgrade to version 2"
$ git push
~~~

Within the ArgoCD dashboard:

* Detect git changes: "Refresh"
* Preview Differences: "App Diff"
* Deploy New Version: "Sync"

## Lab 3: Troubleshoot a degraded app

Within the ArgoCD dashboard:

1. Open app
2. Find the red heart
3. Clik on the resource and check each tab

## Lab 4: An emergency rollback

Within the ArgoCD dashboard:

* Click "History And Rollback"
* Click "..." button previous commit which was successful
* Click "Rollback"
* Click "Ok" in the modal panel

All is good now, but our DC is out-of-sync...

With Git commands, you can could also revert back:

~~~bash
# Revert the commit with ID...
$ git revert $(git rev-parse HEAD)
$ git push
~~~

...and now in our CD tool we are back - next to back in business - also back in sync!
