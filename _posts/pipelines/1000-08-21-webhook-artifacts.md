---
date: 2018-08-21
title: Webhook Stage - Producing Artifacts
categories:
   - Pipeliens
description: Webhook Stage - Producing Artifacts
type: Document
---

The `Webhook Stage` is capable of producing artifacts which can be used in downstream stages. This is done by returning a valid array of artifacts in the response of the webhook that you're calling. For example, your webhook may be responsible for generating an artifact and pushing it into an object store such as S3. To correctly inform Spinnaker of this artifact, your webhook should respond with the following JSON response.

```json
{
    "artifacts": [
        {
            "type": "s3/object",
            "reference": "s3://webhook-example-bucket/webhook-example-object.yaml"
        }
    ]
}
```

In order to use these artifacts within the pipeline, you must first add them as expected artifacts under the pipeline `Configuration` section. 

![](https://cl.ly/2j3U3l3F370M/Image%2525202018-08-21%252520at%25252010.08.24%252520AM.png)


Once you've added a Webhook stage, be sure to add your artifact under the `Produces Artifacts` section in the stage configuration.

![](https://cl.ly/2E292s1v0u0m/Image%2525202018-08-21%252520at%25252010.09.47%252520AM.png)
***

### More Resources: 
- [Artifact Account Types](https://docs.armory.io/install-guide/adding_accounts/#adding-artifact-accounts)
- [Artifacts Overview](https://www.spinnaker.io/reference/artifacts/)
