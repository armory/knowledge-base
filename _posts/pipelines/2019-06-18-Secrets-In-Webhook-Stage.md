---
date: 2019-06-18
title: Secrets in Custom Webhook Stage
categories:
   - Pipelines
description: How to use secrets from S3 in custom webhook stage.
type: Document
---

## Background

Now from [Armory Spinnaker version 2.4.0](https://docs.armory.io/release-notes/armoryspinnaker_v2.4.0/), Managing Spinnaker secrets separately from its configuration is possible which allow us to store our passwords, tokens or sensitive files in secret engines like S3 and Vault, and reference them from config files then halyard can decrypt them when it needs to use them or send them to Spinnaker services to be decrypted upon startup.

This document will show you how to use S3 secrets in a custom webhook stage.  This document also assumes that you have configured your [Spinnaker secrets in S3](https://docs.armory.io/spinnaker-install-admin-guides/secrets-s3/).  If you need instructions on configuring your Spinnaker secrets in Hashicorp’s Vault please refer to [Secrets with Vault](https://docs.armory.io/spinnaker-install-admin-guides/secrets-vault/)

## Store Secrets in S3

First, Let’s store our Api-Key and Payload in S3 `mybucket/spinnaker-secrets.yml`, Amazon Simple Storage Service (Amazon S3) is an object storage service that offers scalability, data availability, security, and performance. Here is [how to enable encryption in S3](https://docs.aws.amazon.com/AmazonS3/latest/dev/bucket-encryption.html).

```yaml
webhook:
  api-key: <API-KEY>
  payload: |-
      {
        "parameters": {
          "user": <USER>,
          "password": <PASSWORD>
        }
      }
```

## Create Custom Webhook Stage

Next, we need to create a custom webhook stage. Following this guide on [Custom Webhook Stages](https://www.spinnaker.io/guides/operator/custom-webhook-stages/), you should add `.hal/default/profiles/orca-local.yml` with the following content: 

```yaml
webhook:
  preconfigured:
  - label: Webhook Secrets S3 
    type: WebhookS3
    enabled: true
    description: Get Secrets in S3
    method: POST
    url: <URI>
    customHeaders:
      x-api-key: encrypted:s3!r:us-west-2!b:mybucket!f:spinnaker-secrets.yml!k:test.api_key
      Content-Type:
        - application/json
    payload: encrypted:s3!r:us-west-2!b:mybucket!f:spinnaker-secrets.yml!k:test.payload
```

Following this guide on [Secrets with S3](https://docs.armory.io/spinnaker-install-admin-guides/secrets-s3/), in order to reference secrets in S3 from our config files, we should follow the next format: `encrypted:s3!r:<region>!b:<bucket>!f:<path to file>!k:<optional key>`

Run `hal deploy apply` and you should see a new orca pod. And if you want to verify you can exec into orca, and find the orca-local.yml file under `/opt/spinnaker/config/`, at this moment Spinnaker will decrypt the secrets in S3, any change in S3 will require run `hal deploy apply`. 

## Add Webhook to a Pipeline

After you've deployed your changes, you should now be able to see the new stage when configuring your pipeline. 
Just select the stage and run the pipeline and the webhook will use the secrets configured in S3.

<a href="https://cl.ly/9ffa17ee081f" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/221f253D391J211x2B0X/Screen%20Shot%202019-06-17%20at%202.59.28%20PM.png" style="display: block;height: auto;width: 100%;"/></a>

In Conclusion, this is just one scenario from many where we can take advantage from this new feature that add security in our config files.