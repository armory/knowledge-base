---
date: 2018-07-20
title: YAML WebHooks Placement in Halyard
categories:
   - Kubernetes
description: Placing YAML webhooks in Halyard to hook to Spinnaker's Orca
type: Document
---

### Question:

*This is a Q&A pulled from the Spinnaker Community Slack team, which [we encourage you to join here](http://join.spinnaker.io).*

Where do I put a custom webhook yaml in halyard for it to get merged into Spinnakers’ Orca?


***

### Answer:

all webhooks go under `webhooks.preconfigured` in `orca-local.yml`.

This should be helpful: https://www.spinnaker.io/guides/operator/custom-webhook-stages/

And here’s how to add a custom configuration using halyard: https://www.spinnaker.io/reference/halyard/custom/

***

### More Resources: 
- [You can find more Kubernetes & Spinnaker related content here](http://go.armory.io/kubernetes)