---
date: 2019-07-18
title: How to create private GKE cluster 
categories:
   - GCP
description: A guide to create private Google Kubernetes Engine cluster with different level of restricted access to master
type: Document
---

### What you will learn?
How to create private kubernetes cluster in Google cloud with different levels of configuration to control access to the cluster master public endpoint

***

### Overview:
Access to Google cloud offers
Having multiple Spinnaker installations, with multiple AWS accounts, share the same configuration repository is possible. First let's cover a few things that need to be baselined, after that we can jump over to our docs for more details.

### Spinnaker Environment:
This is where Spinnaker (the service) lives. Usually correlated with an appropriate DNS entry. ex: https://spinnaker.yourcompany.com. This is useful for different levels of isolations. There's multiple methods of isolations you can do: multiple clouds (Kubernetes, AWS), AWS accounts, Kubernetes namespaces, different VPCs, or even just different instances with new datastores.

### Deployment Target:
This is where Spinnaker is configured to deploy *to*. This is useful for managing applications across different cloud providers, accounts.

### Documentation
With the above understanding jump over to our documentation and enjoy.
[https://docs.armory.io/admin-guides/shared_configuration_repo/](https://docs.armory.io/admin-guides/shared_configuration_repo/)


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


<a href="https://cl.ly/1d686d0f23ed" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/2h1d3d2V3I0C162j0K1t/Image%202019-05-29%20at%201.13.08%20PM.png" style="display: block;height: auto;width: 100%;"/></a>