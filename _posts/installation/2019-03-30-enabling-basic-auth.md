---
date: 2019-03-30
title: Enabling Basic Auth for Spinnaker via Halyard
categories:
   - Installation
description: Enabling Basic Auth for Spinnaker via Halyard
type: Document
---

If you want to enable Simple Auth for Spinnaker using Halyard:
Create/update the `.hal/<deployment-name>/profiles/gate-local.yml` file, with these contents:

```yml
security:
  basicform:
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
