---
date: 2018-07-19
title: Deciding on Manifest Namespaces
categories:
   - Kubernetes
description: Q&A about manifest namespaces
type: Document
---

### Question:

*This is a Q&A pulled from the Spinnaker Community Slack team, which [we encourage you to join here](http://join.spinnaker.io).*

How does the provider decide which namespace to use to apply manifests? Just tried to apply my first one and it created it on the spinnaker namespace. Is that the default?

***

### Answer:

Typically manifests will define the namespace in which they are applied.

If you want to do that dynamically, however, you’ll need to get a bit creative until we have a solution for this: 

https://github.com/spinnaker/spinnaker/issues/2831
GitHub
[Kubernetes V2] Add namespace option for deployManifest · Issue #2831 · spinnaker/spinnaker
Issue Summary: I would like to see an (optional) text field in the UI where I can set the target namespace. With this I won't have to create a deployment manifest for each namespace I deploy to...


***

### More Resources: 
- [You can find more Kubernetes & Spinnaker related content here](http://go.armory.io/kubernetes)