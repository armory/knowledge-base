---
date: 2020-04-17
title: Enabling the Managed Pipeline Templates UI
categories:
   - Admin
description: How to enable the Managed Pipeline Templates UI in Armory Spinnaker
type: Document
---

Armory Spinnaker 2.19 contains the latest version of Managed Pipeline Templates v2 (MPTv2), which is the default pipeline templating solution offered in OSS Spinnaker. 

Armory recommends using Armory's Pipeline as Code feature instead of MPTv2 because it offers the following benefits:

* Integration with GitHub, GitLab and BitBucket enabling teams to store pipelines with application code
* Templates and access to the templates can be stored and managed separately from pipelines
* The ability to compose complex templates and pipelines from modules

Note that Armory's Pipeline as Code and the open source Managed Pipeline Templates are not integrated and do not work together.

By default, the Managed Pipeline Tempalte UI is disabled in Armory Spinnaker 2.19.5. Leaving the UI disabled maintains the same experience you had with Armory Spinnaker 2.18.x (OSS 1.18.x).

If you want to enable the Managed Pipeline Templates UI, run the following Halyard command:

```
hal config features edit --pipeline-templates true
hal config features edit --managed-pipeline-templates-v2-ui true
```

Next, apply your changes to Spinnaker:

```
hal deploy apply
```
