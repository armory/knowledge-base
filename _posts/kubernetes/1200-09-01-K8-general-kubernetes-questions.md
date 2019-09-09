---
date: 2018-04-17
title: Spinnaker Kubernetes V2 FAQs
categories:
   - Kubernetes
description: "Spinnaker with Kubernetes V2"
type: Document
---
04/19/2018

Below we will answer some general questions on Kubernetes (k8s) V2 and how it works with Armory Spinnaker

## How to create a pipeline in Kubernetes V2 with Spinnaker?
Generally speaking, to create a Kubernetes V2 pipeline, you would create a ‘deployment’ manifest for your server group. You can store it in either GitHub, S3, or GCS. In the pipeline you can configure it to trigger when there is a change. Details to each can be found in the OSS docs.

You can find more details [here](https://www.spinnaker.io/reference/providers/kubernetes-v2/#using-externally-stored-manifests).


## How does Spinnaker handle multi-cluster and multi-region deploys to Kubernetes?
In Spinnaker you can configure multiple Kubernetes accounts with different clusters located in different regions, and then each deploy stage can target a specific account. So you can have N number of deploy stages, each going to a different Kubernetes accounts.


## How does Spinnakers Kubernetes V2 handle Istio changes?
There is no extra support for Istio right now. Some things may work perfectly, while others may not work as expected. Armory and the OSS community are still determining how to best fit Istio into the provider.

## How do you handle pipelines that sidecar containers? Are these just defined in the yml?
Yes. You can add any containers you need to the ‘deployment’/’replicaSet’ manifest. This can be initContainers, side cars, etc.

## How do you handle ingress with Let's encrypt SSL certificates?
It is possible to deploy an ingress resource in the same way you can deploy any manifest. An nginx-ingress controller is configured by a config-map, which can be redeployed by Spinnaker to make changes. Since an ingress controller is a daemon deployed as a pod, it can also be redeployed by Spinnaker if desired.

See [here](https://www.spinnaker.io/reference/providers/kubernetes-v2/#services-ingresses) for more information.

## Can V1 kube and V2 kube live on the same cluster?
Yes. However, in the Spinnaker UI you will see every Kubernetes resource twice. Once, when the V1 provider sees it, and once, when the V2 provider sees it.

## Is it possible to apply secrets using the [Base (Future Simple) Helm Secrets Plugin](https://github.com/futuresimple/helm-secrets)?
It is not possible today.

## How do we handle deployment in multiple clusters? For example, if our environment includes more than one Kubernetes cluster.
With the account concept in Kubernetes, an account is just a Kubernetes cluster. When you configure an account you configure it to use an specific kubeconfig file. This defines how we communicate and authenticate on our clusters. If you have multiple clusters you can configure multiple accounts within Spinnaker using different kubeconfig files that have the address for the API. When you set the deployment step you set up a different account to deploy to, and Spinnaker will use the credentials found in the kubeconfig.

## There are two different types of pipelines: Strategy and Pipeline. What's the difference?
A pipeline is generally just the way your application rolls out. That would be like saying deploying to a particular environment Manual Judgment deploying into another, we do things like kicking of Jenkins jobs. Strategy is a way of taking all of the different pieces of that pipeline and being able to select them as a way you want to deploy. 

## It seems that Spinnaker is polling data from Kubernetes. How often is this data refreshed?
By default Spinnaker tries to pull every 30 seconds, but you can configure the interval.

## How's the performance of Spinnaker when it's keeping track of hundreds of pipelines?
Spinnaker was built to run at scale - it is safe to run hundreds of pipelines with Spinnaker. However, that comes with the caveat that you have to manage and scale Spinnaker components. We have a guide at [spinnaker.io](https://spinnaker.io) that describes which components can scale horizontally, depending on your workload. In the case of pipelines, there is a component called Front50, and you can add more replicas to prevent issues with large numbers of pipelines.

## Is the V2 provider still in beta and subject to significant changes?
Release 1.10 sees this provider out of Beta, but there are still many large features that will get added to the provider. For example Traffic Management for Red/Black, Rolling Red/Black, Canary, etc. Also added label manipulation and Istio.

Changes to the Managed Pipeline Templates to allow operators to define custom pipelines that the users can use insted of the users making pipelines by hand over and over again.

Artifact provenance for software supply chain management, this will allow you to ask questions like, which commitment made into production since the last release, what changes in my config map trigger this release.

## Any plans to move from kubectl to Kubernetes APIs?
There are no plans at the moment. One of the reasons we use kubectl, especially for deployment, is that there is some merging logic that is built into kubectl that we are taking advantage of. So the primary way that people interact with the Kubernetes API today is through kubectl.

As it stands using any of the APIs besides the oficial one and kubectl is very dificult and odds are they are largely unsupported.

## Can the manifests be uploaded using a Spinnaker API?
The best practice is to store them outside the pipeline. If you want to do it inline you will not upload the manifests to Spinnaker, what you would do be to upload the pipeline which will contain those manifests. You can upload a task saying, deploy this manifests to the Spinnaker API, if you have the manifests in mind and you have it´s representation, you can send that to Spinnaker to deploy it.

## Or with a pull model, can the dev and prod stages call a service or CLI to pull the appropriate manifest(s) for their env?
The manifest can come from different places, GitHub, GitLab, S3.

## Is Force Cache Refresh duration dependent to the number of objects in manifest of stage? I mean, number of objects in manifest of stage might increase the duration?
The Force Cashe Refresh task in the deploy stage is basically making sure that after it has submitted everything that the Spinnaker orchrestation engine has an aqcurate view of what´s happening on Kubernetes. So Spinnaker tries as much as possible to read from it´s cashe the reason being, with the number of clients that will be looking at Spinnaker UI for example you will have thoundsands and thoundsands of requests duplicated against the server, so everything tries to read at that Cashe as most as possible, the issue of course is updating that Cashe it´s tricky because you have a lot of different compiting mechanismn making sure that things are up to date, so, when you have a lot of manifests it don´t actually add a whole lot of time to Force Cashe Refresh stage, that time is mostly dominated by how many cluster you have or how many resource are in those clusters in the first place when Spinnaker is indexing them because everytime it´s rerhesing the Cashe, it has to look at all the resources in all the cluster and then take any updates that happened like during deployment and the more resources you have the longer that step takes.

## For helm charts usage is there a way to locally test the bake process?
Helm bakes the same way that Spinnaker does it, we just use helm template under the hood, it´s a subcommand within the top level helm command, if you have a chart you want to see how it will be rendered with Spinnaker you can just run that with helm template, you can search more infomation on their page helm.sh.

## Can we use ChartMuseum with Spinnaker instead of putting Helm charts to S3?
Absolutely, ChartMuseum is effectivily just an http server, managing Helm charts is easier but in the end it´s just an http server. As long as you can configure ChartMuseum at an end point you can configure Spinnaker to pull that chart of of ChartMuseum, the main thing that you need to remember is that whaterver is serving your charts just needs to send them back as a tarball.
