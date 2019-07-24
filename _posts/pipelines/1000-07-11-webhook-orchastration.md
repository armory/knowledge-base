---
date: 2018-07-11
title: Using Spinnaker as an Orchestration Tool
categories:
   - Pipelines
description: Using Spinnaker as an Orchestration Tool
type: Document
---

While the primary value that Spinnaker brings to CI/CD workflows is based on deployments, there are stages that make is useful for _other_ things, like orchestration. For example, the `Webhook` stage can be used to make calls into external systems.

![](https://dha4w82d62smt.cloudfront.net/items/0z3V2S1e3c3w2o2x0S1q/Image%202018-07-11%20at%2010.22.33%20AM.png)

As seen in the image above, you can use the `Webhook` stage to perform various requests to systems that Spinnaker doesn't have a direct integration with. For example, your team may have an existing deployment tool that you wish to continue using while you transition to Spinnaker. By stringing these stages together, you can glue multiple systems together to achieve the pipeline that you would otherwise have to script with something like Jenkins. 

By default, the `Webhook` stage assumes that the work being done by the webhook is synchronous and that a successful (`HTTP 2xx`) response indicates a completed action. For some systems, however, work may be done in an ansynchronous fashion which requires polling to ensure successful completion. To do this, you can use the `Wait for completion` option which will allow you to poll an endpoint which will indicate the status of the work triggered by the initial webhook.

![](https://dha4w82d62smt.cloudfront.net/items/3J110t310d1w2A3d0y1y/Image%202018-07-11%20at%2010.28.43%20AM.png)


Using a webhook to trigger events across your stack is useful, but most APIs respond with some useful information that you can use to influence the flow of your pipeline. Using a mix of [SPEL](https://docs.armory.io/user-guides/expression-language/) and conditional checks, we can create pipelines that branch based on the response from our webhook. For example, given a webhook stage (`Call Endpoint`) that responds the payload `{"deployment": {"success": true}}` we can craft a SPEL expression that will evaluate the response. 

```
${#stage('Call Endpoint')['context']['webhook']['body']['deployment']['success'] == true}
```

We can then use that expression as a condition in downstream stages by using the `Conditional on Expression` option. This will dynamically enable or disable stages based on the value returned by the expresion.

![](https://dha4w82d62smt.cloudfront.net/items/0a0T2V1a171N1l2h1i0i/Image%202018-07-11%20at%2010.54.05%20AM.png)

![](https://dha4w82d62smt.cloudfront.net/items/3C3j2z3V3Q1k1T3o2g3U/Image%202018-07-11%20at%2010.55.58%20AM.png)

In the example above, you can see that the `Conditional on False` stage was skipped because the result of our webhook call was `deployment.success == true`.

***

### More Resources: 
- [Custom Webhook Stages](https://www.spinnaker.io/guides/operator/custom-webhook-stages/)
