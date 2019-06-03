---
date: 2019-05-29
title: AWS API Gateway Integration
categories:
   - AWS
description: How to integrate AWS API Gateway to expose Spinnaker webhook
type: Document
---

## Background

There are a handful of options to manage the accessing to Spinnaker by third party systems, without Spinnaker being exposed externally, this document will show you how to expose Spinnaker's resources thorugh AWS API Gateway which will help us to monitor and restrict access to our expose resources.

In this case we will expose a webhook URL to trigger a pipeline which could be called from a third party system outside of our infrestructure, this document also assumes that you have configured Spinnaker in EKS.

## Adding a webhook trigger in Spinnaker

First, we need to add a trigger to our pipeline, under Configuration, select Add Trigger and make its type selector Webhook, this will cause Spinnaker to expose an endpoint that must be hit in order to trigger the pipeline, for more information about configuring a webhook trigger check here.

<a href="https://cl.ly/db231d0a5f88" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/2C3M0l1O3R2L1M1W3h3R/Image%202019-06-03%20at%2011.22.33%20AM.png" style="display: block;height: auto;width: 100%;"/></a>

## Creating AWS API Gateway

Next, we can proceed in the creation of an API in AWS API Gateway, the main intention of this is to maintain security and control over Spinnaker's exposed endpoints, and API Gateway offers an easy way to create, publish, maintain, monitor, and secure APIs at any scale.

<a href="https://cl.ly/f31a2f6bed8b" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/1h3b3H210x0g0j2F1U09/Image%202019-06-03%20at%2011.23.02%20AM.png" style="display: block;height: auto;width: 100%;"/></a>

You might notice after have deployed your API, that establishing a connection between AWS API Gateway to Spinnaker it’s not straight forward, this is mainly because API Gateway runs in its own VPC and is completely managed by AWS, removing the option of VPC peering between both networks. 

## Creating Network Load Balancer

Fortunately, with AWS API Gateway, you can do “Private Integrations”, which is a way to expose your HTTP/HTTPS resources behind an Amazon VPC for access by clients or third party systems outside of the VPC, currently only Network Load Balancers are supported to make private integrations.

Now, let create a Network load balancer in our EKS cluster with kubernetes service, remember that Spinnaker expose webhook URLs through Gate, which is Spinnaker’s API Gateway, all traffic flows through Gate.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: spin
    cluster: spin-gate
  name: spin-gate-internal
  namespace: default
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8084
  selector:
    app: spin
    cluster: spin-gate
  sessionAffinity: None
  type: LoadBalancer
```

You might notice the annotations aws-load-balancer-type: "nlb" and aws-load-balancer-internal: "true" that will cause AWS to create an internal-facing Network load balancer when we create this service in EKS.

## Creating VPC Link

Now, in API Gateway we can create a VpcLink to encapsulate connections between API Gateway and targeted VPC resources, just we need to select our NLB prior created.

<a href="https://cl.ly/3098e92af5e7" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/3d0A3G1M1r36231U1m2C/Image%202019-06-03%20at%2011.23.20%20AM.png" style="display: block;height: auto;width: 100%;"/></a>

## Adding endpoint to API Gateway

At this point, we have our API Gateway with private integration to our private EKS cluster well configured, now just we need to add our webhook URL or other resources that we want to expose externally via API.

<a href="https://cl.ly/dd4adf81212f" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/051F310V0Y1v0j031M0k/Image%202019-06-03%20at%2011.23.44%20AM.png" style="display: block;height: auto;width: 100%;"/></a>

Now we can take advantage of all benefits that API Gateway offers in the moment to expose our resources, as access by IAM policies, api-keys, IP range, and more, without changing our EKS cluster's network configuration.