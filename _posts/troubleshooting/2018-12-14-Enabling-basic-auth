---
date: 2018-12-14
title: Enabling Basic Auth using Halyard
categories:
   - troubleshooting
description: Enabling basic or simple Auth using Spinnaker.
type: Document
---

If you want to enable Simple Auth using Spinnaker: 
Create/update the `.hal/<deployment-name>/profiles/gate-local.yml` file, with these contents:

```
security:
  basic:
    enabled: true
  user:
    name: <username you want>
    password: <password you want>
```
And then create/update the `.hal/<deployment-name>/profiles/settings-local.js`, with these contents

```
window.spinnakerSettings.authEnabled = true;
```

Then, apply your change (`hal deploy apply`).
