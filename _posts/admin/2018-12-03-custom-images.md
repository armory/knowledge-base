---
date: 2018-12-03
title:  Using custom images with Halyard
categories:
    - Admin
description:  Using custom images with Halyard
type: Document
---

Occasionally, it may make sense to update the Spinnaker Kubernetes deployments created by Halyard with a custom Docker image.  This can be done through a custom service setting.

Here is how you would achieve this: identify the service that you're modifying, and create a corresponding file in `.hal/<deployment-name>/service-settings/<service-name>.yml` with the relevant artifactId.

For example, if I want to use a custom Docker image for my Deck, I could create the following file:

#### `.hal/<deployment-name>/service-settings/deck.yml`
```yml
artifactId: docker.io/armory/deck:<some-other-tag>
```

Then, when you go to deploy your update (`hal deploy apply`), this setting should propagate to `.hal/<deployment-name>/staging/spinnaker.yml`, in `services.deck.artifactId`, and get deployed to the cluster.

More generically, individual settings in `spinnaker.yml` can be overridden with files in `.hal/<deployment-name>/service-settings/<service-name>.yml`, and additional yaml files can be propagated to containers by placing them in `.hal/<deployment-name>/profiles/<service-name>-local.yml`