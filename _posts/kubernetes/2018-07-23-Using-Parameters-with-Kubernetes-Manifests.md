---
date: 2018-07-19
title: Using Parameters with Kubernetes Manifests
categories:
   - Kubernetes
description: Q&A about manifest namespaces
type: Document
---

### Question:

*This is a Q&A pulled from the Spinnaker Community Slack team, which [we encourage you to join here](http://join.spinnaker.io).*

How does the provider decide which namespace to use to apply manifests? Just tried to apply my first one and it created it on the `spinnaker` namespace. Is that the default?

***

### Answer:

[spinnaker/spinnaker#2831](https://github.com/spinnaker/spinnaker/issues/2831) defines a way we could make this more dynamic within the `Deploy (Manifest)` stage.

Until that issue is closed you can use Pipeline parameters and SPEL within your manifests as seen [here](https://kb.armory.io/kubernetes/parameters-with-Kubernetes/).


***

### More Resources: 
- [You can find more Kubernetes & Spinnaker related content here](https://kb.armory.io/kubernetes/parameters-with-Kubernetes/)
- [spinnaker/spinnaker#2831](https://github.com/spinnaker/spinnaker/issues/2831)