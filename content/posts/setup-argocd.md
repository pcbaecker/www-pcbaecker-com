---
title: "How to setup ArgoCD to manage itself"
date: 2022-01-09T00:00:00Z
draft: false
---

ArgoCD is in my opinion one of the best GitOps-Tools available. It checks a given Git-Repo and applies the configuration from there to a kubernetes cluster. The setup described in the ArgoCD-Guide using a single .yaml file is easy and straight forward. But from there you have to manually update and control ArgoCD whereas the rest of software will be controlled by ArgoCD. To also have ArgoCD update and configure itself it not much harder.

## Install the helm chart

Attention: The name of the helm chart must be the same as the name of the argocd application we create later! I just use "argocd" for both.

The first step is to setup ArgoCD with Helm, make sure to specify the correct namespace (I use [Lens IDE](https://k8slens.dev/) to manage helm charts but that makes no difference):

{{< highlight bash >}}
helm repo add argo https://argoproj.github.io/argo-helm
helm install --namespace argocd argo argo/argo-cd
{{< / highlight >}}

After all pods are running we have to login to the UI by forwarding the argoCD service:

{{< highlight bash >}}
kubectl port-forward svc/argocd-server -n argocd 9090:443 --address=0.0.0.0
{{< / highlight >}}

There you should see a login screen with the username 'admin' and the generated password given by:

{{< highlight bash >}}
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
{{< / highlight >}}

## Create the git repo with the App of Apps

In the next step we have to create a new git repo containing the application description for ArgoCD. We use the 'App of Apps'-pattern, which means that we create an app that contains the other ArgoCD apps that should be created. With that method we have to create only one app manually and all other apps will be pulled from the git repo. Therefore create a folder calles 'apps' containing two files:

***Chart.yaml***
{{< highlight yaml >}}
apiVersion: v2
name: applications
description: Applications
type: application
version: 0.1.0
appVersion: "1.0"
{{< / highlight >}}

***values.yaml***
{{< highlight yaml >}}
spec:
  destination:
    server: https://kubernetes.default.svc
  source:
    repoURL: git@github.com:hpcbaecker/example-argocd.git
    targetRevision: HEAD
{{< / highlight >}}

and then a folder 'apps/templates' containing:

***argocd.yaml***
{{< highlight yaml >}}
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: 'https://argoproj.github.io/argo-helm'
    targetRevision: 3.29.5
    chart: argo-cd
    helm:
      values: |
        server:
          extraArgs:
            - "--disable-auth"
            - "--insecure"
          ingress:
            enabled: "true"
            hosts:
              - argocd.local
            annotations:
              # kubernetes.io/ingress.class: ambassador ### For some ingress controllers required
        dex:
          enabled: "false"
  destination:
    server: {{ .Values.spec.destination.server }}
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
{{< / highlight >}}

## Connect ArgoCD to the git repo

As the next step we have to add our git repo. To do so click on 'Settings' -> 'Repositories' -> 'Connect repo using SSH'. To authenticate against the git repo we need a ssh keypair [(can be created here)](https://8gwifi.org/sshfunctions.jsp). Github has recently made some changes to their ssh-key requirements, therefore I recommend to use their guide for ssh-keys [Github SSH-Keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) For some reason the server verification is not working for me, so we skip that.

![Add a new git repo](/static/images/setup-argocd-add-repo.png)

## Create the App

After that we have to create the (master) application containing all the real apps we want to be managed. Therefore we go to 'Applications' -> 'New App'.

![Add a new git repo](/static/images/setup-argocd-create-the-app.png)

After that one is created a few moments later argocd will be synced to the git repo and manage itself. New software can be creating by adding .yaml files in 'apps/templates' with the application structure from argocd.yaml. When commited ArgoCD will pull the changes and apply e.g. the new helm chart to the cluster.

## A few words about this ArgoCD config

In the given config above I disabled https connections and authentication. This is very useful for evalutation and maybe local servers but should never be used in production. Also the ArgoCD-CLI may not work without a proper TLS connection.
