---
date: 2019-05-13
title: Long force cache refresh (Kubernetes)
categories:
   - troubleshooting
description: Long force cache refresh (Kubernetes)
type: Document
---
A long Kubernetes manifest deploy, scale, or undo rollout stage, and more specifically a `Force Cache Refresh` task that lasts 12 minutes is a sign of Orca being unable to acknowledge the cache refresh that occurred in Clouddriver:

![](/images/2c16cfed7cf87c8d1ad2558beeb1e0b0.png)


## Solution
Open your pipeline definition (Pipeline Actions > Edit As JSON) and make sure that the first word of the `manifestName` attribute in stage definitions is all lower case (`deployment` in the example below):

![](/images/ea7b33b49b95caf57dfb2ef343ce65d3.png)

If it's not, correct the `manifestName` and save your changes.

## Explanation
Several stages require that the target manifest be updated in the cache and do it via a `Force Cache Refresh` task. The `manifestName` is a representation of the manifest you're targeting that Orca will use when inspecting cached values (from Clouddriver). It is made of the Kubernetes kind and the manifest name. Earlier versions of Spinnaker were incorrectly saving the kind as is.