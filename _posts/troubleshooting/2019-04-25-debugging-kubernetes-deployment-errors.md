---
date: 2019-04-25
title: Debugging Deployment Errors
categories:
   - troubleshooting
description: Troubleshooting deployment errors such as `deployKubernetesManifest.deployKubernetesManifest.deployment.Error`
type: Document
---

## Debugging your Kubernetes Deployment Errors

When deploying to a new Kubernetes cluster for the first time, you might receive this error: 
`deployKubernetesManifest.deployKubernetesManifst.deployment.Error reading kind [deployment].`

Typically this is caused by one of the following issues:  
1. Your kubeconfig for the cluster is not properly configured.  Check this doc on creating a unique kubeconfig for the cluster with the proper roles.  [Adding Kubernetes Account](https://docs.armory.io/spinnaker-install-admin-guides/add-kubernetes-account/)
  a. To verify, try running `kubectl --kubeconfig [yourUniqueKubeconfig] get ns` to see you can access the new cluster. (You can run the command from any system)

2. The namespace used in the deployment manifest does not match the namespaces allowed as defined in your .hal/config.  The K8s cluster defined in your .hal/config needs to have the namespaces white-listed.
You can add the namespaces by running (from within Halyard) `hal config provider kubernetes account edit [theNewClusterAccountName] --add-namespace [TargetNamespaceName]

For additional troubleshooting: 
Check the CloudDriver logs for additional information: `kubectl -n spinnaker logs -f spin-clouddriver-*****`


