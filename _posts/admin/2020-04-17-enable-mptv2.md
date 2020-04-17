---
date: 2020-04-17
title: Enabling Managed Pipeline Templates
categories:
   - Admin
description: How to enable the Managed Pipeline Templates feature in Armory Spinnaker
type: Document
---

Armory Spinnaker 2.19 contains the latest version of Managed Pipeline Templates, which is the default pipeline templating solution offered in OSS Spinnaker. This feature is disabled by default in Armory Spinnaker.

Instead, Armory recommends using Armory's Pipeline as Code feature instead because it offers the following benefits:

* Integration with GitHub, GitLab and BitBucket enabling teams to store pipelines with application code
* Access controls can be stored and defined for templates separately from pipelines
* The ability to compose complex templates and pipelines from modules

Note that Armory's Pipeline as Code and the open source Managed Pipeline Templates are not integrated and do not work together.

If you want to enable Managed Pipeline Templates, add the following configuration to `SOME HAL CONFIG FILE`:

```
<insert instructions to enable>
```
