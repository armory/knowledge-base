---
date: 2020-01-01
title: Flushing Gate Sessions
categories:
   - Admin
description: How to flush Gate's HTTP sessions
type: Document
---

## Background

Gate depends on Sprint Boot and Spring Session to manage its HTTP sessions. When Gate authenticates a user, this session is serialized and stored in Redis.

This serialization is dependent on  the versions of Spring Boot and Spring Session. 

When there is a new version of those dependencies, users encounter the following error when they try to log in:

```
HTTP Status 500 - Internal Server Error
```

Additionally, Deck redirects the browser to the Gate URL.
<br>
To resolve the issue, perform one of the following actions:
1. Remove the browser cookie for all users.
2. Flush all the stored Gate sessions and force users to log in again

While option 1 is simple, Armory recommends option 2 for deployments where there are a large number of users.


## Flushing Gate sessions

### Requirements
You must have network access to Redis and have the `redis-cli` installed. If you do not have the `redis-cli` already installed, Step 1 includes installing it.
<br><br>

### Step 1. Install and configure the debugging tool

Armory recommends using the [`armory/docker-debugging-tools`](https://github.com/armory/docker-debugging-tools) and deploying it into your Spinnaker namespace. This image contains `redis-cli`. Run the following commands:

```bash
kubectl --context $MY_CONTEXT -n $MY_NAMESPACE apply -f https://raw.githubusercontent.com/armory/troubleshooting-toolbox/master/docker-debugging-tools/deployment.yml

kubectl --context $MY_CONTEXT -n $MY_NAMESPACE exec -it deploy/debugging-tools -- bash
```

### Step 2. Find your Redis URL:

Run the following command to exec into the Gate pod: 

```bash
k --context $MY_CONTEXT -n $MY_NAMESPACE exec -it deploy/spin-gate -c gate -- cat /opt/spinnaker/config/spinnaker.yml | grep -A2 redis
```

Capture the Redis url from the `baseUrl` field in the resulting output, it will look something like the follwoing:

```
  redis:
    baseUrl: redis://my-redis-url.example.com:6379
    enabled: true
    host: 0.0.0.0
```

In this example, you would use `my-redis-url.example.com`, without the protocol and port data.

Then, run the following command in the pod you started in your cluster to flush the sessions:

```bash
redis-cli -h my-redis-url.example.com keys 'spring:session:*' | xargs redis-cli del
```

_Note: you may safely ignore errors connecting to 127.0.0.1:6379 if your redis
is located elsewhere._

### Step 3. Remove the debugging tools

When you are finished, you can remove the debugging tools by running the following command:

```bash
kubectl --context $MY_CONTEXT -n $MY_NAMESPACE delete deployment debugging-tools
```


## Extra Resources

[spinnaker/issues/1947](https://github.com/spinnaker/spinnaker/issues/1947)

[spring-session/issues/280](https://github.com/spring-projects/spring-session/issues/280)
