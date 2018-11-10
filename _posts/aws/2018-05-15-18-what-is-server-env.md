---
date: 2018-05-15
title: What is /etc/default/server-env?
categories:
   - AWS
description: "What is /etc/default/server-env?"
type: Document
---

### Question:
- What is `/etc/default/server-env`?
- How does a running instance know what application or environment it's in?
- How does `/opt/spinnaker/env/ha.env` or `/opt/spinnaker/env/prod.env` get sourced during runtime?
- How does `udf` work?

***

### Answer:
When [udf is enabled for aws](https://www.spinnaker.io/setup/features/user-data/#aws) for example, Spinnaker will inject server information into the **user data** during an execution of a deploy stage of an application. This information will end up in into `/etc/default/server-env`.
If we have an application called `armoryspinnaker-prod-nonpolling`, it'll look something like:
```bash
$ cat /etc/default/server-env
CLOUD_ACCOUNT="aws-prod"
CLOUD_ACCOUNT_TYPE="aws-prod"
CLOUD_ENVIRONMENT="aws-prod"
CLOUD_SERVER_GROUP="armoryspinnaker-prod-nonpolling-v984"
CLOUD_CLUSTER="armoryspinnaker-prod-nonpolling"
CLOUD_STACK="prod"
CLOUD_DETAIL="nonpolling"
EC2_REGION="us-west-2"
LAUNCH_CONFIG="armoryspinnaker-prod-nonpolling-v984-0828050116281"
```

***

### More Resources: 
#### UDF configuration
[https://docs.armory.io/admin-guides/userdata/](https://docs.armory.io/admin-guides/userdata/)


#### UDF template
[https://www.spinnaker.io/setup/features/user-data/](https://www.spinnaker.io/setup/features/user-data/)
