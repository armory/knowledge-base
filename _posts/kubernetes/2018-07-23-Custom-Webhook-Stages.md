---
date: 2018-07-20
title: Custom Webhook Stages
categories:
   - Kubernetes
description: Placing YAML webhooks to hook to Spinnaker's Orca
type: Document
---

### Question:

*This is a Q&A pulled from the Spinnaker Community Slack team, which [we encourage you to join here](http://join.spinnaker.io).*

Where do I put a custom webhook yaml in halyard for it to get merged into Spinnakers’ Orca?


***

### Answer:

All webhooks go under `webhooks.preconfigured` in `orca-local.yml`.

You can find documentation on defining custom webhook stages [here](https://www.spinnaker.io/guides/operator/custom-webhook-stages/).

If you're using OSS Spinnaker, you'll need to add a custom Orca profile as seen [here](https://www.spinnaker.io/reference/halyard/custom/).

***

### More Resources: 
- [You can find more Kubernetes & Spinnaker related content here](https://blog.spinnaker.io/custom-spinnaker-stages-with-preconfigured-webhooks-84c5b5dae861)