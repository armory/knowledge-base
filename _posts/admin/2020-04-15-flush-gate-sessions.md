---
date: 2020-01-01
title: Flushing Gate Sessions
categories:
   - Admin
description: How to flush Gate's http sessions
type: Document
---

## Background

Gate depends on springboot and spring-session to manage it's HTTP sessions. When Gate authenticates a user, this session is serialized and stored into Redis.

The issue is that the serialization is very dependent on versions of springboot or spring-session. 

When a new version of those dependencies change for gate, we'll need to:
1. have the user remove their cookie, however this is not scalable, therefore option 2.
2. flush all the stored sessions and force all users to login again

Your users will see an error like this when trying to login:
<kbd> 
  <img src="/images/5651fe80-7f4f-11ea-bc45-43e571ce330c.png" /> 
</kbd>

Notice that the url is the API, i.e. Gate, not Deck's. If you were doing option 1, you must clear the cookies from Gate's URL after Deck has redirected you to login.



## The Fix: Flushing Gate Sessions
You'll first need to get network access to Redis and have the `redis-cli` installed.

A quick tip is to use [`armory/docker-debugging-tools`](https://github.com/armory/docker-debugging-tools) to be deployed into your spinnaker namespace. This image contains `redis-cli`. The command is copied here for easy of use:
```bash
kubectl --context $MY_CONTEXT -n $MY_NAMESPACE apply -f https://raw.githubusercontent.com/armory/docker-debugging-tools/master/deployment.yml

kubectl --context $MY_CONTEXT -n $MY_NAMESPACE exec -it deploy/debugging-tools -- bash
```

Now to find your Redis url:
```bash
k --context $MY_CONTEXT -n $MY_NAMESPACE exec -it deploy/spin-gate -c gate -- cat /opt/spinnaker/config/spinnaker.yml | grep -A2 redis
```

Then inside the pod, you can run:
```bash
redis-cli -h my-redis-url.armory.io keys 'spring:session:*' | xargs redis-cli del
```

When you're all finished, you can remove the image by doing:
```bash
kubectl --context $MY_CONTEXT -n $MY_NAMESPACE delete deployment debugging-tools
```


## Extra Resources
[spinnaker/issues/1947](https://github.com/spinnaker/spinnaker/issues/1947)

[spring-session/issues/280](https://github.com/spring-projects/spring-session/issues/280)