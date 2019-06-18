---
date: 2019-05-29
title: AWS S3 Secrets in Custom Webhook Stages
categories:
   - AWS
description: How to use secrets from S3 in custom webhook stages.
type: Document
---

## Background

Early this year from [version 1.15.0](https://github.com/spinnaker/halyard/releases), Halyard supports secret management that allow us to store our passwords even files as kube configs in S3 and reference them from a yml file, this is a great feature that we can use in several scenarios where we want to hide information and store them in a safe place.

This document will show you how to use S3 secrets in a custom webhook stage.  This document also assumes that you have configured an AWS account.  If you need more instructions on configuring your AWS account please refer to either [Deploying to AWS from Spinnaker (using IAM instance roles)](https://docs.armory.io/spinnaker-install-admin-guides/add-aws-account-iam/) or [Deploying to AWS from Spinnaker (using IAM credentials)](https://docs.armory.io/spinnaker-install-admin-guides/add-aws-account/) 


## Store Secrets in S3

First, Let’s store our Api-Key and Payload in S3 `mybucket/spinnaker-secrets.yml`, Amazon Simple Storage Service (Amazon S3) is an object storage service that offers scalability, data availability, security, and performance. Here is [how to enable encryption in S3](https://docs.aws.amazon.com/AmazonS3/latest/dev/bucket-encryption.html)

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

Run `hal deploy apply` and you should see a new orca pod. And if you want to verify you can exec into orca, and find the orca-local.yml file under `/opt/spinnaker/config/`, at this moment Spinnaker will decrypt the secrets in S3, any change in S3 will require run `hal deploy apply` 

## Add Webhook to a Pipeline

After you've deployed your changes, you should now be able to see the new stage when configuring your pipeline. 
Just select the stage and run the pipeline and the webhook will use the secrets configured in S3.

<a href="https://cl.ly/9ffa17ee081f" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/221f253D391J211x2B0X/Screen%20Shot%202019-06-17%20at%202.59.28%20PM.png" style="display: block;height: auto;width: 100%;"/></a>

In Conclusion, this is just one scenario from many where we can take advantage from this new feature that add security in our config files.