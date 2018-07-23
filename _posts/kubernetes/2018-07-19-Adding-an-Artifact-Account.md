---
date: 2018-07-19
title: Adding an Artifact Account
categories:
   - Kubernetes
description: Q&A about adding an artificat account on s3
type: Document
---

### Question:

*This is a Q&A pulled from the Spinnaker Community Slack team, which [we encourage you to join here](http://join.spinnaker.io).*

I have spent almost half a day trying to add an Artifact Account just like it shows on yours as `s3`. I can only see `embedded-artifact` in my pipeline and errors out as
```
\"message\":\"500 \",\"body\":\"{\\\"error\\\":\\\"Internal Server Error\\\",\\\"exception\\\":\\\"java.lang.IllegalArgumentException\\\",\\\"message\\\":\\\"Artifact credentials 'embedded-artifact' cannot handle artifacts of type
``` 

***

### Answer: 

If you're using Armory Spinnaker, add an S3 Artifact account to `clouddriver-local.yml` using [this config](https://docs.armory.io/install-guide/adding_accounts/#s3).

If you're using OSS Spinnaker, you can add an S3 account using [these Halyard commands](https://www.spinnaker.io/reference/halyard/commands/#hal-config-artifact-s3). 


***

### More Resources: 
- [You can find more Kubernetes & Spinnaker related content here](https://docs.armory.io/install-guide/adding_accounts/#adding-artifact-accounts)
- [And here](https://www.spinnaker.io/reference/artifacts/)

