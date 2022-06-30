---
title: "Github actions selfhosted runner on kubernetes"
date: 2022-06-27T00:00:00Z
draft: false
---

Using [Github](https://github.com/)'s CI tool [Github Actions](https://github.com/features/actions) can be very useful for projects that are already hosted on [Github](https://github.com/). For public available projects you get unlimited build minutes and thats amazing. But for private repositories the number of build minutes is limited to the owners account. Even though 2000 build minutes are included in the free account plan, you might exceed this by having many repositories. Having a self hosted runner will give you an infite amount of build minutes.

More relevant can the self hosted runner be when you have specific build requirements such as building on M1 Apple Silicon, optimized for aarch64 or such. With the self hosted runner you have more control over OS and machine.

# Setup cert manager

One requirement for the [Github Self Hosted Runner](https://github.com/actions-runner-controller/actions-runner-controller) is a kubernetes project called "cert manager". It can be easily installed as a helm chart, however I prefer using [ArgoC](https://argoproj.github.io/cd/) to manage my helm charts:

***certmanager.yaml***
{{< highlight yaml >}}
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: certmanager
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: 'https://charts.jetstack.io'
    targetRevision: 1.8.2
    chart: cert-manager
    helm:
      values: |
        installCRDs: "true"    
  destination:
    server: {{ .Values.spec.destination.server }}
    namespace: certmanager
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
{{< / highlight >}}

# Creating the Github Personal Access Token

In order to authenticate the self hosted runner against the github servers we need to issue a personal access token. You can do that at [https://github.com/settings/tokens](https://github.com/settings/tokens). You now have to decide whether the self hosted runner should be repository specific or organization specific. 

For repository specific runner it is sufficient to give the "repo" scope.

![New Personal Access Token Repo](/static/images/github-selfhosted-runner-on-kubernetes-pat-repo.png)

If you want a self hosted runner for the whole organization you have to additionally grant the "admin:org" scope.

![New Personal Access Token Organization](/static/images/github-selfhosted-runner-on-kubernetes-pat-orga.png)

# Creating the controller

Now we have to create the [Github Actions Runner Controller](https://github.com/actions-runner-controller/actions-runner-controller). The easiest way is by using a helm chart. I use ArgoCD for me to handle that. In the helm chart values we must provide the Personal Access Token generated at the previous step.

***github-actions-runner-controller.yaml***
{{< highlight yaml >}}
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: github-selfhosted-runner
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: 'https://actions-runner-controller.github.io/actions-runner-controller'
    targetRevision: 0.19.1
    chart: actions-runner-controller
    helm:
      values: |
        authSecret:
          github_token: ghp_T69u...
          create: true 
  destination:
    server: {{ .Values.spec.destination.server }}
    namespace: github-selfhosted-runner
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
{{< / highlight >}}

After the controller pod is booted up the log should show the current status as authenticated as follows:

    xxx TODO

It can happen, that the CRDs are not correctly created by ArgoCD or Helm. There is a workaround [here](/posts/kubernetes-crd-annotation-too-long-fix/).

# Creating the runner

The controller can now launch [Github Actions Runners](https://github.com/actions-runner-controller/actions-runner-controller#organization-runners) described in the CRDs. The easiest way is to create a runner deployment. The .yaml files for repository specific and organization specific differ by the spec attribute.

***repo-runner-deployment.yaml***
{{< highlight yaml >}}
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: repo-runner-deployment
spec:
  replicas: 1
  template:
    spec:
      repository: my-username/my-repositoryname
{{< / highlight >}}

***orga-runner-deployment.yaml***
{{< highlight yaml >}}
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: orga-runner-deployment
spec:
  replicas: 1
  template:
    spec:
      organization: your-organization-name
{{< / highlight >}}

You can create the fitting deployment.yaml by:

    kubectl apply -f repo-runner-deployment.yaml

After that you should see a pod starting in kubernetes and after a while the self hosted runner will be visible on one of the following links:

***https://github.com/organizations/your-organization/settings/actions/runners***
***https://github.com/your-username/your-reponame/settings/actions/runners***

# Use the self hosted runner

To make your github action run on your self hosted runner you have to set the "runs-on" attribute to "self-hosted".

***.github/workflows/my-action.yaml***
{{< highlight yaml >}}
// ...
jobs:
  build:
    runs-on: self-hosted
    steps:
    // ...
{{< / highlight >}}