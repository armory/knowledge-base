---
date: 2018-04-17
title: Spinnaker Kubernetes V2 FAQs
categories:
   - Kubernetes
description: "Spinnaker with Kubernetes V2"
type: Document
---
04/19/2018

Below we will answer some general questions on Kubernetes (K8s) V2 and how it works with Armory Spinnaker

## How to create a pipeline in Kubernetes V2 with Spinnaker?
Generally speaking, to create a Kubernetes V2 pipeline, you would create a ‘deployment’ manifest for your server group. You can store it in either GitHub, S3, or GCS. In the pipeline you can configure it to trigger when there is a change to it. Details to each can be found in the OSS docs.

You can find more details at this link:
[https://www.spinnaker.io/reference/providers/kubernetes-v2/#using-externally-stored-manifests](https://www.spinnaker.io/reference/providers/kubernetes-v2/#using-externally-stored-manifests)


## How does Spinnaker handle multi-cluster and multi-region deploys to Kubernetes?
In Spinnaker you can configure multiple Kubernetes accounts with different clusters which are located in different regions and then each deploy stage can target a specific account. So you can have N number of deploy stages, each going to a different kubernetes accounts.


## How does Spinnakers Kubernetes V2 handle ISTIO changes?
There is no extra support for ISTIO right now. Some things may work perfectly, while others may not work as expected. Armory and the OSS community still determining how to best fit ISTIO into the provider.

## How do you handle pipelines that sidecar containers? Are these just defined in the yml?
Yes. You can add any containers you need to the ‘deployment’/’replicaSet’ manifest. This can be initContainers, side cars, etc.

## How do you handle ingress with Let's encrypt SSL certificates?
It is possible to deploy an ingress resource in the same way you can deploy any manifest. An nginx-ingress controller is configured by a config-map, which can be redeployed by Spinnaker to make changes. Since an ingress controller is a daemon deployed as a pod, it can also be redeployed by Spinnaker if desired.

[https://www.spinnaker.io/reference/providers/kubernetes-v2/#services-ingresses](https://www.spinnaker.io/reference/providers/kubernetes-v2/#services-ingresses)

## Can V1 kube and V2 kube live on the same cluster?
Yes. However, in the Spinnaker UI you will see every K8s resource twice. Once, when the V1 provider sees it, and once, when the V2 provider sees it.

## is it possible to apply secrets using https://github.com/futuresimple/helm-secrets?
Today that is not possible

## How do we handle deployment in multiples cluster? For example, if our environment includes more than one k8s cluster
With the account concept in K8s, an account is just a k8s cluster, when you configure an account you configure it to use an specific kube config file, this define how we communicate and authenticate on our cluster, if you have multiple cluster you can configure multiple accounts within spinnaker using different kube config files that have the address for the API, when you set the deployment step you setup a different account to deploy to and spinnaker will use the credentials found in the kube config.

## There are two different types of pipelines : Startegy and Pipeline , what's the difference.
A pipeline is generally just the way your application rolls out, that would be like saying deploying to a particular environment manual judgment deploying into another, we do things like kicking of jenkins jobs. Strategy is kind of a way of taking all of the different pieces of that pipeline and being able to select them as a way you want to deploy. 

## It seems that Spinnaker is polling data from Kubernetes. How often is this data refreshed?
By default spinnaker tries to pull every 30 seconds, but you can configure this time.

## How's the performance of Spinnaker when it's keeping track of hundreds of pipelines?
Spinnaker was build to run at scale, hundreds of pipelines are safe for spinnaker but comes with the caviat that you have at some extend manage and scale spinnaker controlers. We have at spinnaker.io a little guide at what compponents to scale horizontall, depending on how your work looks like, in the case of pipelines there is a component called front50, you have to probably add more replicas of that so you don´t run into issues when running large number of pipelines

## is the V2 provider still in beta and subject to significant changes?
Since release 1.10 it's out of Beta, there are still many large features that will get added to the provider. For example Traffic Management for Red/Black, Rolling Red/Black, Canary, etc. Also added label manipulation and Istio.
Changes to the Managed Pipeline Templates to allow operators to define custom pipelines that the users can use insted of the users making pipelines by hand over and over again.
Artifact provenance for software supply chain management, this will allow you to ask questions like, which commitment made into production since the last release, what changes in my config map trigger this release.

## Any plans to move from kubectl to K8S APIs?
There is no plans at the moment, one of the reasons we use kubectl specially for deployment is that there is some merging loginc that is build into kubectl that we are taking advantage of, so the primary way that the people interact with the k8s API today is through kubectl.
As it stand using any of the apis beside the oficial one and kubectl is very dificult and odds they are largely unsupported

## Can the manifests be uploaded using a Spinnaker API?
The best practice is to store them outside the pipeline. If you want to do it online you will not upload the manifest to spinnaker, what you would do be to upload the pipeline which will contain those manifest. You can upload a task saying, deploy this manifest to the spinnaker API, if you have the manifest in mind and you have it´s representation, you can send that to spinnaker to deploy it.

## Or with a pull model, can the dev and prod stages call a service or CLI to pull the appropriate manifest(s) for their env?
The manifest can come from different places, GitHub, GitLab, S3

## Is Force Cashe Refresh duration dependent to the number of objects in manifest of stage? I mean, number of objects in manifest of stage might increase the duration?
The Force Cashe Refresh task in the deploy stage is basically making sure that after it has submitted everything that the spinnaker orchrestation engine has an aqcurate view of what´s happening on K8s. So spinnaker tries as much as possible to read from it´s cashe the reason being, with the number of clients that will be looking at spinnaker UI for example you will have thoundsands and thoundsands of requests duplicated against the server, so everything tries to read at that Cashe as most as possible, the issue of course is updating that Cashe it´s tricky because you have a lot of different compiting mechanismn making sure that things are up to date, so, when you have a lot of manifest it don´t actually add a whole lot of time to Force Cashe Refresh stage, that time is mostly dominated by how many cluster you have or how many resource are in those clusters in the first place when spinnaker is indexing them because everytime it´s rerhesing the Cashe, it has to look at all the resources in all the cluster and then take any updates that happened like during deployment and the more resources you have the longer that step takes.

## For helm charts usage is there a way to locally test the bake process?
Helm bakes the same way that spinnaker does it, we just use helm template under the hood, it´s a subcommand within the top level helm command, if you have a chart you want to see how it will be rendered with spinnaker you can just run that with helm template, you can search more infomation on their page helm.sh

## Can we use ChartMuseum with Spinnaker instead of putting Helm charts to S3?
Absolutely, ChartMuseum is effectivily just an http server, managing Helm charts is easier but in the end it´s just an http server. As long as you can configure ChartMuseum at an end point you can configure spinnaker to pull that chart of of ChartMuseum, the main thing that you need to remember is that whaterver is serving your charts just needs to send them back as a tarball.
