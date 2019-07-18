---
date: 2019-07-16
title: AWS CloudFormation Example
categories:
   - AWS
description: How to enable CloudFormation and deploy a simple template.
type: Document
---

## Background

This document will show you how to enable the CloudFormation stage.  This document also assumes that you have configured an AWS account.  If you need more instructions on configuring your AWS account please refer to either [Deploying to AWS from Spinnaker (using IAM instance roles)](https://docs.armory.io/spinnaker-install-admin-guides/add-aws-account-iam/) or [Deploying to AWS from Spinnaker (using IAM credentials)](https://docs.armory.io/spinnaker-install-admin-guides/add-aws-account/) 

## Preconditions
&nbsp;
### Enable Cloudformation in Spinnaker

First, we need to enable CloudDriver by adding a `clouddriver-local.yml` file to your hal config profiles directory e.g. `.hal/default/profiles/clouddriver-local.yml`.
```yaml
aws:
  features:
    cloudFormation:
      enabled: true
```
You can check your configuration by running `hal deploy apply` and check the clouddriver logs (e.g. `kubectl logs -f -n spinnaker spin-clouddriver-xxxxx`) for any start-up errors.  Refer to the [Debugging](#debugging) section for checking your deployment.

&nbsp;
### Halconfig 

Make sure to enable the aws provider and have set at least one region

```yaml
aws:
  enabled: true
  accounts:
  - name: aws-1
    requiredGroupMembership: []
    providerVersion: V1
    permissions: {}
    accountId: '123456789123'
    regions:
    - name: us-east-1
    - name: us-west-2
    assumeRole: role/SpinnakerManagedRole
  primaryAccount: aws-1
  bakeryDefaults:
    templateFile: aws-ebs-shared.json
    baseImages: []
    awsAssociatePublicIpAddress: true
    defaultVirtualizationType: hvm
  defaultKeyPairTemplate: '-keypair'
  defaultRegions:
  - name: us-west-2
  defaults:
    iamRole: BaseIAMRole
```

&nbsp;
### Policy

Another thing to check is the `cloudformation:*` permission in the *"Action"* section inside your **Policy**, like in the following example. 

_for more details please refer to [Creating a Managing Account IAM Policy in your primary AWS Account](https://docs.armory.io/spinnaker-install-admin-guides/add-aws-account/#iam-user-part-3-creating-a-managing-account-iam-policy-in-your-primary-aws-account)_

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::123456789123:role/SpinnakerManagedRole"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "ec2:*",
                "cloudformation:*"
            ],
            "Resource": "*"
        }
    ]
}
```

## Creating your Pipeline

After you've deployed your changes, you should now be able to see the new stages when configuring your pipeline. 
Simply select the stage, and provide the values: example below: 

![](/images/cloudformation-pipeline1.png)

In this exaple we are going to deploy a simple s3 bucket with the follow template, (Copy and paste in the **Text Source** section)

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Outputs": {
    "BucketName": {
      "Description": "S3 Bucket",
      "Value": {
        "Ref": "S3Bucket"
      }
    }
  },
  "Resources": {
    "S3Bucket": {
      "Properties": {
        "BucketName": "cf-example-s3"
      },
      "Type": "AWS::S3::Bucket"
    }
  }
}

```

## Run the pipeline

Now you can run the pipeline and this will be conclude **SUCCEEDED**
![](/images/cloudformation-success.png)

At the meanwhile you can see the execution details if you go to your *AWS CloudFormation console* 
![](/images/cloudformation-aws-console.png)


## Validate
In this particular example we created an S3 bucket so in order to validate you need to go to your aws console and check in the S3 bucket section.
![](/images/cloudformation-s3.png)
