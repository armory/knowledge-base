---
date: 2020-01-01
title: Restrict Application Creation
categories:
   - Admin
description: How to restrict application creation in Spinnaker with Fiat Permissions
type: Document
---

## Background

Before version 2.17, there was no way to prevent application creation in Spinnaker, now in 2.17, Fiat add a new option which allow achieve this, with a new permission option CREATE.

This document assumes that you have enabled and configured Fiat.

## Restrict application creation

**This was tested on version 2.17 and may change in further versions.**

Fiat is the authorization (authz) microservice of Spinnaker, which look for the permissions from different sources. In 2.17 a new source was added which is more flexible and applies permissions to any application whose name starts with a given prefix, `auth.permissions.source.application.prefix`.

By default, Fiat only reads from the legacy sources, to change this and allow Fiat reads from other sources like the prefix one, we need to change the value to `aggregate` of the parameter `auth.permissions.provider.application`.

```yaml
#fiat-local.yml
auth.permissions.provider.application: aggregate
auth.permissions.source.application.prefix:
  enabled: true
  prefixes:
   - prefix: "apptest-*"
     permissions:
       READ:
       - "role-one"
       - "role-two"
       WRITE:
       - "role-one"
       EXECUTE:
       - "role-one"
```

This will apply the READ restriction on all apps starting with the prefix `apptest-*` to role-two, and add READ, WRITE and EXECUTE permissions to role-one.

Now, to prevent the application creation in Spinnaker, we have a new option in Fiat to enable `fiat.restrictApplicationCreation` which will allow us to set the CREATE permission to the roles.

**Currently the prefix source is the only source that support the CREATE permission.**

```yaml
#fiat-local.yml
fiat.restrictApplicationCreation: true
auth.permissions.provider.application: aggregate
auth.permissions.source.application.prefix:
  enabled: true
  prefixes:
   - prefix: "*"
     permissions:
       CREATE:
       - "role-one"
       READ:
       - "role-one"
       - "role-two"
       WRITE:
       - "role-one"
       EXECUTE:
       - "role-one"
```
This will prevent the creation of any application to users without CREATE permission, only the users with role-one will create applications in Spinnaker.

<a href="https://cl.ly/0381358c5f72" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/2p0o1i0g0w30203c303s/Screen%20Shot%202020-01-03%20at%2011.47.51.png" style="display: block;height: auto;width: 100%;margin-bottom:10px"/></a>

