---
date: 2018-12-03
title: Incorrect OAuth2.0 Redirects
categories:
   - troubleshooting
description: Incorrect OAuth2.0 Redirects
type: Document
---

If you've set up OAuth2.0 authentication for your Spinnaker cluster and are seeing redirects to the wrong page (for example, to `http` instead of `https`), try the following:

```bash
hal config security authn oauth2 edit --pre-established-redirect-uri https://my-real-gate-address.com:8084/login
```

This will modify your `.hal/config` with this field:
```yml
  security:
    authn:
      oauth2:
        client:
          preEstablishedRedirectUri: https://my-real-gate-address.com:8084/login
```

Additionally, add/create `.hal/<deployment-name>/profiles/gate-local.yml`:

```yml
server:
  tomcat:
    protocolHeader: X-Forwarded-Proto
    remoteIpHeader: X-Forwarded-For
    internalProxies: .*
```

This is also documented here: https://www.spinnaker.io/setup/security/authentication/oauth/#network-architecture-and-ssl-termination