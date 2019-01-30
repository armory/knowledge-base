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

#### Basic Math
Taking the number of current instances of a given Deployment and scaling it by some non-integer value, where `Get Deployment` is a "Find Artifacts from Resource (Manifest)" (`findArtifactsFromResource`) stage that looks at a Kubernetes `deployment` object:

```java
${ (0.8 * #stage("Get Deployment")["outputs"]["manifest"]["spec"]["replicas"]).intValue()}
```

#### Using JSON and #readJson to create parameter aliases
1) create a parameter `environment` with 3 options (`development`, `staging`, `production`)
2) create a parameter `short_env_name` with the default value of `${#readJson('{"development": "dev", "staging": "stag", "production":"prod"}')}`

When you need to short name, you can just reference it like so `${parameters.short_env_name[parameters.environment]}`


#### Using a ternary operator to condition something on current state:
Ternary operator:
```java
<some-condition> ? <value-if-true> : <value-if-false>
```

Simple example:
```java
${ true ? "True text" : "False text" }
```


Example (in the text of a Manual Judgement stage), where `Get Service` is a "Find Artifacts From Resource (Manifest)" (`findArtifactsFromResource`) that looks at a Kubernetes `service` object:
```
The loadBalancer ingress is ${ #stage("Get Service")["outputs"]["manifest"]["status"]["loadBalancer"].containsKey("ingress") ? "ready" : "not ready" }.
```