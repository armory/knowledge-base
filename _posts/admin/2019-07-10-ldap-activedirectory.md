---
date: 2019-07-10
title: Using Active Directory for Authentication
categories:
   - Admin
description: How to setup AD for LDAP authentication (authorization is a separate topic)
type: Document
---

### Question:

What settings do I need to use to use AD for LDAP authentication?

***

### Answer:

AD does some things slightly differently than other LDAP servers, but it's very viable to setup!  Here's how to do so....

### Step-by-step Instructions
Note, these instructions came after debugging an issue and finding information from an issue ticket [Authorization WIndows Active directory LDAP](https://github.com/spinnaker/spinnaker/issues/4417#issuecomment-494871659) courtesy of Git user ryanwoodsmall.

First, make sure you're using [Halyard](https://github.com/spinnaker/halyard) greater than 1.6.0 as that adds the ability to do manager settings in gate. You can use a prior version, but you'd need to use a gate-local.yml to define the properties vs. in the halyard config file.  

For reference, here's the settings you're looking for in a yaml config file
```yaml
ldap:
  enabled: true
  url:  ldaps://somethingsomething:686/ou=users,ou=company,o=com
  userSearchFilter: (&(objectClass=user)(sAMAccountName={0}))
  managerDn: CN=SVC_LDAP_USER_RO,OU=service-users,OU=company,O=com
  managerPassword: super-secret-password
```

<b>Do NOT use the userDnPattern</b>.  Per issue comment: "userDnPattern should remain unset - AD groups store user DNs in the memberOf attribute; finding DNs from sAMAccountNames is easily doable but not with a simple, single-level pattern. The DN contains the the CN, and that can't really be constructed without sub searches. userSearchFilter takes precedence if there's no userDnPattern set."

The managerUser will then find a user in your ou=users,ou=company,o=com directory via a subtree search.  You should be able to set the user-search-base parameter vs. including it on the URL to have it specified separately.  

Last, here's the halyard commands to configure these:
```
hal config security authn ldap enable
hal config security authn ldap edit \
  --manager-dn 'CN=blah,OU=blah,OU=blah,O=blah' \
  --user-search-filter '(&(objectClass=user)(sAMAccountName={0}))' \
  --url ldaps://blah:686/searchbase
## This one will prompt you for the password don't set it on the command
hal config security authn ldap edit --manager-password
hal deploy apply

```

## Escaping
IF your CN/DN have odd characters, don't worry about it!  Spring LDAP will automatically replace most of the standard characters (comma, backslash) with the correct LDAP escape sequences.  Put the value in as it exactly is in your directory.
