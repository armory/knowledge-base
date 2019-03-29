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

## Basic Math
Take the number of instances for a deployment and multiply by a non-integer value:<br/>
Note: "Get Deployment" is the name of the "Find Artifacts from Resource (Manifest)" stage that looks at a Kubernetes deployment object.<br/>

```java
${ (0.8 * #stage("Get Deployment")["outputs"]["manifest"]["spec"]["replicas"]).intValue() }
```

## <a name="helper_properties"></a>Helper Properties

Create a parameter `environment` with 3 options:

![environment parameter](https://cl.ly/792c433f0efa/Image%2525202019-03-28%252520at%2525202.06.04%252520PM.png)
<br/>
Access the value of the variable via a helper property, in this case, `parameter`. For example:<br/>

```java
The environment is: ${ parameters.environment }
```

Some Helper Properties are already defined, for example:

```java
The execution id is automatically set: ${execution['id']}
```

## Helper Functions

Read and print a JSON, for example:

```java
${#readJson('{"development": "dev", "staging": "stage", "production":"prod"}').toString()}
```

<br/>
You can also access a value in the JSON with the `environment` parameter from the [Helper Properties](#helper_properties) Section:

```java
${#readJson('{"development": "dev", "staging": "stage", "production":"prod"}')[parameters.environment]}
```

## Ternary Operator

```java
<some-condition> ? <value-if-true> : <value-if-false>
```

<br/>
Simple example:

```java
${ true ? "True text" : "False text" }
```

<br/>
Example (in the text of a Manual Judgement stage), where `Get Service` is a "Find Artifacts From Resource (Manifest)" that looks at a Kubernetes `service` object:

```java
The loadBalancer ingress is ${ #stage("Get Service")["outputs"]["manifest"]["status"]["loadBalancer"].containsKey("ingress") ? "ready" : "not ready" }.
```

## Whitelisted Java Classes

Some Java classes are available for use in SpEL expressions (see [Spinnaker Reference Docs](https://www.spinnaker.io/reference/pipeline/expressions/#whitelisted-java-classes))
<br/>
For example, generating the current date in MM-dd-yyyy format:

```java
${new java.text.SimpleDateFormat("MM-dd-yyyy").format(new java.util.Date())}
```

<br/>
Sometimes you may want to call a static method. For example, to generate a UUID:

```java
 ${T(java.util.UUID).randomUUID() }
```
