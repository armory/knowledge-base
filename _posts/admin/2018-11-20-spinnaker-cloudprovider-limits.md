---
date: 2018-11-20
title:  Spinnaker Cloud Provider Limit
categories:
   - Admin
description:  Spinnaker Cloud Provider Limit
type: Document
---

### Question:

Is there a limit to the number of Kubernetes clusters that one can define in Spinnaker?

***

### Answer:

Nope!  There isn't a hard limit to the number of Kubernetes (or other) clusters you can define in Spinnaker.  Depending on the number of clusters and resources you're running, you may need to scale out multiple clouddrivers to handle caching.

***
