---
date: 2018-08-01
title: Migrate Clouddriver's Redis To Its Own Instance
categories:
   - Admin
description: As you begin to scale your usage of Spinnaker, it makes sense to start separating out your redis instances per micro-service
type: Document
---

### Steps To Move An Exising Cluster:

1. Take a snapshot of your existing Redis cluster
1. Create a new cluster with the snapshot from the previous step.  This will include all the keys from the previous cluster that you can clear out later.
1. Configure Clouddriver to use the new cluster.  In your `/opt/spinnaker/config/clouddriver-local.yml` add the following:
```yml
redis:
  connection: redis://${YOUR_REDIS_HOST}:6379
```
1. Redeploy your Spinnaker instance