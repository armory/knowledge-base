---
date: 2019-01-30
title: SpEL Tricks
categories:
   - Pipelines
description: Spinnaker Pipeline expression examples, tips, and tricks
type: Document
---

Spinnaker uses the Spring Expression Language (SpEL) for pipeline expressions, so you can do a lot of interesting things with Spinnaker expressions.

Here are a couple of the more interesting examples that we've come across (this page will grow):

#### Using JSON and a Map to shorten variable names
1) create a parameter `environment` with 3 options (`development`, `staging`, `production`)
2) create a parameter `short_env_name` with the default value of `${#readJson('{"development": "dev", "staging": "stag", "production":"prod"}')}`

when you need to short name, you can just reference it like so `${parameters.short_env_name[parameters.environment]}`


#### Using a ternary operator to condition something on current state:
Ternary operator:
```java
<some-condition> ? <value-if-true> : <value-if-false>
```

Example (in the text of a Manual Judgement stage):
```
You can check the status of the application here: <a href='${ #stage("Get Ingress")["outputs"]["manifest"]["status"]["loadBalancer"].containsKey("ingress") ?
"http://" + #stage("Get Ingress")["outputs"]["manifest"]["status"]["loadBalancer"]["ingress"][0]["hostname"] + "/api" : "https://spinnaker.sales.armory.io/#/applications/demoapp/loadBalancers"}'>Demo App</a>.
```