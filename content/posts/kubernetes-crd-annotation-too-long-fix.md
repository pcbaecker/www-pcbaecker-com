---
title: "Fix for kubernetes crd annotation too long problem"
date: 2022-06-24T19:00:00Z
draft: false
---

Sometimes creating custom resource definitions (CRD) in kubernetes can be a bit challenging. Even with the help of helm charts there you may run in the following error while creating CRDs:

    CustomResourceDefinition.apiextensions.k8s.io "runnerdeployments.actions.summerwind.dev" is invalid: metadata.annotations: Too long: must have at most 262144 bytes

Tools like ArgoCD or HELM normally use "kubectl apply -f ..." to apply a yaml file. For CRDs with often very long description texts this can lead to the given error. The problem is, that an annotation "kubectl.kubernetes.io/last-applied-configuration" will be created and that one is too large.

# Fix

To fix the problem you can just use the following commands instead of "kubectl apply -f ...":

    kubectl create -f ...  // For creating the resource
    kubectl replace -f ... // For updating the resource