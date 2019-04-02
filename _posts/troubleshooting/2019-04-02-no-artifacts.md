---
date: 2019-03-29
title: No "Expected Artifacts" Field
categories:
   - troubleshooting
description: No "Expected Artifacts" Field
type: Document
---

If you've installed Spinnaker and are trying to configure artifacts, but don't see the "Expected Artifacts" field in the configuration tab of your Spinnaker pipeline, make sure you have artifacts enabled:

```bash
hal config features edit --artifacts true

hal deploy apply
```