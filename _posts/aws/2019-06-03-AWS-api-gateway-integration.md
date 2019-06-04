---
date: 2019-05-29
title: AWS API Gateway Integration
categories:
   - AWS
description: How to integrate AWS API Gateway to expose Spinnaker webhook
type: Document
---

## Background

There are a handful of options to manage the accessing to Spinnaker by third party systems, without Spinnaker being exposed externally, this document will show you how to expose Spinnaker's resources thorugh [AWS API Gateway](https://aws.amazon.com/api-gateway/) which will help us to monitor and restrict access to our expose resources.

In this case we will expose a webhook URL to trigger a pipeline which could be called from a third party system outside of our infrestructure, this document also assumes that you have configured Spinnaker in EKS.

## Adding a webhook trigger in Spinnaker

First, we need to add a trigger to our pipeline, under Configuration, select Add Trigger and make its type selector Webhook, this will cause Spinnaker to expose an endpoint that must be hit in order to trigger the pipeline, for more information about configuring a webhook trigger check [here](https://www.spinnaker.io/guides/user/pipeline/triggers/webhooks/) .

<a href="https://cl.ly/ccd8e0a71a61" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/0L0k2G382g1H092V3f2W/Screen%20Shot%202019-06-03%20at%203.52.56%20PM.png" style="display: block;height: auto;width: 100%;"/></a>

## Creating AWS API Gateway

Next, we can proceed in the creation of an API in AWS API Gateway, the main intention of this is to maintain security and control over Spinnaker's exposed endpoints, and API Gateway offers an easy way to create, publish, maintain, monitor, and secure APIs at any scale.

<a href="https://cl.ly/e9d0ae7fb024" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/1X3z1w2A392o3l0L2C2U/Screen%20Shot%202019-06-03%20at%203.17.58%20PM.png" style="display: block;height: auto;width: 100%;margin-bottom:10px"/></a>

You might notice after have deployed your API, that establishing a connection between AWS API Gateway to Spinnaker it’s not straight forward, this is mainly because API Gateway runs in its own VPC and is completely managed by AWS, removing the option of VPC peering between both networks. 

## Creating Network Load Balancer

Fortunately, with AWS API Gateway, you can do [“Private Integrations”](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-private-integration.html), which is a way to expose your HTTP/HTTPS resources behind an Amazon VPC for access by clients or third party systems outside of the VPC, currently only Network Load Balancers are supported to make private integrations.

<a href="https://cl.ly/02e7d06c8a7a" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/3D082j3U1u431F1k2L1y/Screen%20Shot%202019-06-03%20at%2011.15.54%20PM.png" style="display: block;height: auto;width: 75%;"/></a>

Before create a Network load balancer,  Let's create a NodePort that opens a specific port on all worker nodes, and any traffic that is sent to this port will be forwarded to the Gate's service.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spin-gate-internal
  namespace: default
  labels:
    app: spin
    cluster: spin-gate
spec:
  type: NodePort
  selector:
    app: spin
    cluster: spin-gate
  ports:
  - name: http
    port: 8084
    protocol: TCP
```

Now, let create a Network load balancer in AWS, first select internal scheme, we don't want expose all Gate's endpoints, NLB supports TCP and TLS protocols, select TLS if you want add a certificate from AWS Certificate Manager, it is important to create the NLB in the same VPC of your EKS Cluster.

<a href="https://cl.ly/a324ff1d0243" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/1X2I1B0m1o2l1E013702/Screen%20Shot%202019-06-03%20at%2011.09.32%20PM.png" style="display: block;height: auto;width: 100%;margin-bottom:10px"/></a>

After that, we are going to create a target group which is used to route requests to the registered instances in this group under the protocol and port that you specify, in this case we will use the NodePort created previously and register all EKS's worker nodes.

<a href="https://cl.ly/d864843ecda2" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/2q3z2C1s0e071V121s0b/Screen%20Shot%202019-06-03%20at%2011.13.05%20PM.png" style="display: block;height: auto;width: 100%;"/></a>

## Creating VPC Link

Now, in API Gateway we can create a VpcLink to encapsulate connections between API Gateway and targeted VPC resources, just we need to select our NLB prior created.

<a href="https://cl.ly/a7de58b9ee30" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/0G453A2j2Q102q2J2p2B/Screen%20Shot%202019-06-03%20at%203.20.45%20PM.png" style="display: block;height: auto;width: 100%;"/></a>

## Adding an endpoint to API Gateway

At this point, we have our API Gateway with private integration to our private EKS cluster well configured, now just we need to add our webhook URL or other resources that we want to expose externally via API.

<a href="https://cl.ly/ac14e59a5758" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/1m3d0m111n3r1w0n0u1K/Screen%20Shot%202019-06-03%20at%203.33.20%20PM.png" style="display: block;height: auto;width: 100%;margin-bottom:10px"/></a>

Now we can take advantage of all benefits that API Gateway offers in the moment to expose our resources, as access by IAM policies, api-keys, IP range, and more, without changing our EKS cluster's network configuration.