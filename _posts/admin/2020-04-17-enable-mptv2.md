---
date: 2020-04-17
title: Enabling Managed Pipeline Templates
categories:
   - Admin
description: How to enable the Managed Pipeline Templates UI in Armory Spinnaker
type: Document
---

Armory Spinnaker 2.19 contains the latest version of Managed Pipeline Templates, which is the default pipeline templating solution offered in OSS Spinnaker. By default, the UI is disabled in Armory Spinnaker 2.19 to maintain the same experience as Spinnaker 2.18x (OSS 1.18.x).

Instead, Armory recommends using Armory's Pipeline as Code feature because it offers the following benefits:

* Integration with GitHub, GitLab and BitBucket enabling teams to store pipelines with application code
* Templates and access to the templates can be stored and managed separately from pipelines
* The ability to compose complex templates and pipelines from modules

Note that Armory's Pipeline as Code and the open source Managed Pipeline Templates are not integrated and do not work together.

If you want to enable Managed Pipeline Templates UI, add the following configuration to `SOME HAL CONFIG FILE`:

```
<insert instructions to enable>
```
