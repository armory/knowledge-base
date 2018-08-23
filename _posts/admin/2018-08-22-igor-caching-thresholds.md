---
date: 2018-08-22
title: Hitting Igor's caching thresholds
categories:
   - Admin
description: Igor has caching limits to safeguard against accidental re-indexing 
type: Document
---

## Symptoms
- Completing a Jenkins job doesn't trigger a pipeline execution
- Pushing to a docker repository doesn't trigger a pipeline execution
- In your Igor logs, you see something like:

```
Number of items (999999) to cache exceeds upper threshold (1000) in monitor=DockerMonitor partition=dockerhub
```

or

```
Number of items (999999) to cache exceeds upper threshold (1000) in monitor=JenkinsMonitor partition=jenkins
```


## Causes
Potential causes could be:
- A new Jenkins master has been added
- A new docker registry has been added
- Igor has been down for a while
- Redis has been wiped


## Solution
There are two ways to resolve this issue:
- If this is a one time occurrence (adding a new Jenkins master/new docker repository) then you can use the fastfoward API to advance the pointer.

```bash
curl -X POST http://igor_address_port/admin/pollers/fastforward/dockerTagMonitor
curl -X POST http://igor_address_port/admin/pollers/fastforward/jenkinsBuildMonitor

# To find available monitors, try "help"
curl -X POST http://igor_address_port/admin/pollers/fastforward/help
{
  "message": "PollingMonitor help was not found, available monitors are: [dockerTagMonitor, jenkinsBuildMonitor]",
  "exception": "com.netflix.spinnaker.kork.web.exceptions.NotFoundException",
  "error": "Not Found",
  "status": 404,
  "timestamp": "2018-08-23T00:35:15.592+0000"
}
```

- If this happens often, then you may want to change limits to something higher. In your `igor-local.yml` add:

```yml
spinnaker.pollingSafeguard.itemUpperThreshold: 1000  # some number bigger than your current limits
```