---
date: 2019-03-29
title: Authentication Timeout Results in 401 Error
categories:
   - troubleshooting
description: SAML authentication timeout results in 401 error
type: Document
redirect_from:
  - /troubleshooting/okta-timeout-401/
  - /troubleshooting/2019-03-29-okta-timeout-401/
---
You may encounter the following error when a user attempts to login:
```
{
  error: "Unauthorized",
  message: "Authentication Failed: Error validating SAML message",
  status: 401,
  timestamp: 1553109495710
}
```
<br>
This issue occurs because of a known issue in Spring. Spinnaker does not accept SAML tokens signed by a user who authenticated more than 2 hours ago even if your authentication system allows it.<br><br>

For Armory Spinnaker versions 2.3.0 or later, you can set the maximum authentication age for Spinnaker to match the age you specify for your authentication system.
<br><br>

Configure the following property in your `profiles/gate-local.yml` file at the root level:

```yaml
saml:
  maxAuthenticationAge: <time in seconds for authentication age timeout>
```
