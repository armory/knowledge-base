---
date: 2018-12-12
title: Credential not found error
categories:
   - troubleshooting
description: Credential not found error
type: Document
---

## Error

When deploying to an AWS account you receive the following error:

```
credential not found (name: default, accounts: [accountA, accountB, accountC...])
```

## Solution

This can occur for installations that are deploying into AWS but not using Rosco to bake images. If this occurs it is because you do not have a default bake account configured. This will not enable baking but will prevent the error above. To fix this error, add the following snippet to `~/.hal/default/profiles/clouddriver-local.yml`

```
default:
  bake:
    account: {primary-aws-account-name}
```

By default, the value of `default.bake.account` is `default` which is where the value in the error is coming from. Spinnaker is attempting to fetch credentials for an account that doesn't exist or isn't configured.

