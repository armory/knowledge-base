---
date: 2019-03-14
title: Configuring Authentication using GitHub 
categories:
   - Admin
description: Configuring Spinnaker to use Github (OAuth2) as an authenticator (authn)
type: Document
---

This post will show you how to configure GitHub and Spinnaker to use GitHub as an OAuth2 authenticator.
_This post assumes you have the ability to modify developer settings for your GitHub organization and that you have access to Halyard.  And that you've already configured SSL and URL endpoints for your Spinnaker instance._

### Configuring GitHub OAuth
[https://help.github.com/en/articles/authorizing-oauth-apps]

1. Login to GitHub and go to Settings > Developer Settings > OAuth Apps > New OAuth App
2. Note the Client ID / Client Secret (We will use this later within Halyard)
3. Homepage URL: This would be the url of your Spinnaker service e.g. https://spinnaker.acme.com
4. Authorization callback URL: This is going to match your `--pre-established-redirect-uri` in halyard and the url needs the `login` path appended to your gate endpoint e.g. https://gate.spinnaker.acme.com/login  or https://spinnaker.acme.com/gate/login


### Configuring Spinnaker w/ Halyard

In Halyard run the following commands with your Client ID and Client Secret.

```shell
CLIENT_ID=a08xxxxxxxxxxxxx93
CLIENT_SECRET=6xxxaxxxxxxxxxxxxxxxxxxx59
PROVIDER=github

hal config security authn oauth2 edit \
  --client-id $CLIENT_ID \
  --client-secret $CLIENT_SECRET \
  --provider $PROVIDER \
  --scope read:org,user:email \
  --pre-established-redirect-uri "https://gate.spinnaker.acme.com/login"

hal config security authn oauth2 enable
```