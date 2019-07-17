---
date: 2019-03-13
title: Authenticating against Jenkins
categories:
   - Installation
description: Authenticating against Jenkins
type: Document
---

If you have are trying to authenticate Spinnaker against a Jenkins master that has some third party authentication set up, you have to create an API token for Spinnaker to authenticate against Jenkins.  This is how to do this:

1. Log into Jenkins
2. Click on your username (in the top right)
3. Click on "Configure" (on the left)
4. Under the "API Token" section, click on "Add new Token", and "Generate" and record the generated token
5. Record your username (should be in the URL for the current page - if you're at https://jenkins.domain.com/user/justinrlee/configure, then the username is `justinrlee`)
6. Add the Jenkins master to Spinnaker with this:

    ```
    hal config ci jenkins enable
    echo <TOKEN> | hal config ci jenkins master add <JENKINS-NAME> \
        --address https://jenkins.domain.com \
        --username <USERNAME> \
        --password

        # Password will be read from stdin
    ```

    For example, if the generated token is `1234567890abcdefghijklmnopqrstuvwx`,  and your username is `justinrlee` and you want to identify the Jenkins master as `jenkins-prod`, then you'd run something like this:
    ```
    echo 1234567890abcdefghijklmnopqrstuvwx | hal config ci jenkins master add jenkins-prod \
        --address https://jenkins.domain.com \
        --username justinrlee \
        --password
    ```

Also, don't forget to configure the CSRF stuff: https://www.spinnaker.io/setup/ci/jenkins/#configure-jenkins-and-spinnaker-for-csrf-protection
