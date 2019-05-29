---
date: 2019-05-29
title: AWS Lambda & Custom Webhook Stages
categories:
   - AWS
description: How to enable AWS Lambda and use a custom webhook stage to update your Lambda code.
type: Document
---

## Background

Back in December of 2018, AWS added supported for Lambda and it was released in Spinnaker OSS v1.12.  The challenge, however, was that there was no UI (Deck) components to support the usage of Lambda.  Instead, a [README.md](https://github.com/spinnaker/clouddriver/blob/master/clouddriver-lambda/README.md) was published that specified the API to make changes to Lambda.

This document will show you how to create a custom webhook stage that utilizes the Lambda API built into Clouddriver.  This document also assumes that you have configured an AWS account.  If you need more instructions on configuring your AWS account please refer to either [Deploying to AWS from Spinnaker (using IAM instance roles)](https://docs.armory.io/spinnaker-install-admin-guides/add-aws-account-iam/) or [Deploying to AWS from Spinnaker (using IAM credentials)](https://docs.armory.io/spinnaker-install-admin-guides/add-aws-account/) 

## Enable AWS Lambda in Spinnaker

First, we need to enable Lambda by adding a `clouddriver-local.yml` file to your hal config profiles directory e.g. `.hal/default/profiles/clouddriver-local.yml`
(As of Halyard OSS version 1.20, there is no support for adding Lambda configurations via hal commands.)
```yaml
aws:
  lambda:
    enabled: true
  accounts:
    - name: aws
      lambdaEnabled: true
      providerVersion: V1
      accountId: '555692138000'
      regions:
      - name: us-east-1
      - name: us-west-2
      assumeRole: role/awaylambda05242019-managedrole
```
You can check your configuration by running `hal deploy apply` and check the clouddriver logs (e.g. `kubectl logs -f -n spinnaker spin-clouddriver-xxxxx`) for any start-up errors.  Refer to the [Debugging](#debugging) section for checking your deployment.

## Adding Custom Webhook Stage

Next, we need to add a custom webhook stage.  Alternatively, you could use a simple webhook stage.
Following this guide on [Custom Webhook Stages](https://www.spinnaker.io/guides/operator/custom-webhook-stages/), you should add `.hal/default/profiles/orca-local.yml` with the following content: 

```yaml
webhook:
  preconfigured:
  - label: Lambda - Get Functions
    type: lambdaGetFunctions
    enabled: true
    description: Get Lambda Functions
    method: GET
    url: http://spin-clouddriver:7002/functions
    customHeaders:
      Accept:
        - "application/json"
  - label: Lambda - Update Function Code
    type: lambdaUpdateFunctionCode
    enabled: true
    description: Update Lambda Function Code
    method: POST
    url: http://spin-clouddriver:7002/aws/ops/updateLambdaFunctionCode
    customHeaders:
      Accept:
        - "application/json"
      Content-Type:
        - "application/json"
    payload: |-
      {
        "credentials": "${#root['parameterValues']['account']}",
        "region": "${#root['parameterValues']['region']}",
        "functionName": "${#root['parameterValues']['functionName']}",
        "s3Bucket": "${#root['parameterValues']['bucketname']}",
        "s3Key": "${#root['parameterValues']['key']}",
        "publish": "${#root['parameterValues']['publish']}"
      }
    parameters:
      - label: Spinnaker Account Name
        name: account
        type: string
      - label: Region
        name: region
        type: string
      - label: Function Name
        name: functionName
        type: string
      - label: S3 Bucket Name
        name: bucketname
        type: string
      - label: S3 Key
        name: key
        type: string
      - label: Publish
        name: publish
        type: string
  - label: Lambda - Update Function Configuration
    type: lambdaUpdateFunctionConfig
    enabled: true
    description: Update Lambda Function Configuration
    method: POST
    url: http://spin-clouddriver:7002/aws/ops/updateLambdaFunctionConfiguration
    customHeaders:
      Accept:
        - "application/json"
      Content-Type:
        - "application/json"
    payload: |-
      {
        "region": "${#root['parameterValues']['region']}",
        "functionName": "${#root['parameterValues']['functionName']}",
        "description": "${#root['parameterValues']['description']}",
        "credentials": "${#root['parameterValues']['account']}",
        "role": "${#root['parameterValues']['roleARN']}",
        "timeout": "${#root['parameterValues']['timeout']}"
      }
    parameters:
      - label: Region
        name: region
        type: string
      - label: Function Name
        name: functionName
        type: string
      - label: Description
        name: description
        type: string
      - label: Spinnaker Account Name
        name: account
        type: string
      - label: Role ARN
        name: roleARN
        type: string
      - label: Timeout (secs)
        name: timeout
        type: string
```
You might notice that the `parameterValues` are being referenced with a `#root` helper function. This is to help ensure that Orca can evaluate the expressions using the parameter values from within the stage.

Run `hal deploy apply` and you should see a new orca pod.  And if you want to verify you can exec into orca, and find the orca-local.yml file under `/opt/spinnaker/config/`

## Creating your Pipeline

After you've deployed your changes to orca, you should now be able to see the new stages when configuring your pipeline. 
Simply select the stage, and provide the values: example below: 

<a href="https://cl.ly/1d686d0f23ed" target="_blank"><img src="https://d2ddoduugvun08.cloudfront.net/items/2h1d3d2V3I0C162j0K1t/Image%202019-05-29%20at%201.13.08%20PM.png" style="display: block;height: auto;width: 100%;"/></a>

## Referencing values from `Get Functions`

You can use SPeL to get values from the response to the `Lambda - Get Functions` stage.  Here is an example SPeL expression to get a function name. 

`${#stage("Lambda - Get Functions").context.webhook.body[0].functionName}`

Check the 'source' of the pipeline to see the contents of the webhook - or reference the Get Functions output below in the [curl](#curl) section.
## Debugging 

For debugging your aws lambda setup, check the clouddriver logs: e.g. `kubectl logs -f -n spinnaker spin-clouddriver-xxxxx`
For debugging your custom webhook, check the logs for both orca and clouddriver.

### Debugging Tools 
You can deploy a [debugging pod](https://github.com/armory/docker-debugging-tools) into the same cluster and namespace where Spinnaker lives.  This pod contains useful tools that are missing in the Spinnaker micro-services to help you debug your deployment. Check out the [debugging-tools repo](https://github.com/armory/docker-debugging-tools).

### curl 
With the debugging pod deployed, you can exec into the pod and run the following calls to test if Lambda is properly configured in Spinnaker. The key point to note here is that the base URL is pointing to clouddriver (e.g. http://spin-clouddriver:7002)

```bash
curl -X GET --header 'Accept: application/json' 'http://spin-clouddriver:7002/functions' | jq .
```
This command requests for the lambda functions in the AWS account and pipes the output to jquery to create a nice human readable format.

You should expect an output similar to the following: 
```json
{
   "account": "aws",
   "codeSha256": "gxyL5FY9DLxavA+MMBAFrsnhL7SRL/CiSwciLGGWBCI=",
   "codeSize": 262,
   "description": "",
   "eventSourceMappings": [],
   "functionArn": "arn:aws:lambda:us-west-2:555692138000:function:helloArmoryfx",
   "functionName": "helloArmoryfx",
   "handler": "index.handler",
   "lastModified": "2019-05-28T15:12:40.330+0000",
   "layers": [],
   "memorySize": 128,
   "region": "us-west-2",
   "revisionId": "4a799ab5-9cf2-491c-a07f-129247a27b4e",
   "revisions": {
   "4a799ab5-9cf2-491c-a07f-129247a27b4e": "$LATEST"
   },
   "role": "arn:aws:iam::555692138000:role/service-role/helloArmoryfx-role-y9j7d2ll",
   "runtime": "nodejs10.x",
   "timeout": 10,
   "tracingConfig": {
   "mode": "PassThrough"
   },
   "version": "$LATEST"
}
```
From here you can test other commands such as Lambda update code.

```bash
curl -X POST \
  http://spin-clouddriver:7002/aws/ops/updateLambdaFunctionCode \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
    "region": "us-west-2",
    "functionName": "helloArmoryfx",
    "credentials": "aws",
    "s3Bucket": "armory-sales-away",
    "s3Key": "lambdacode/lambdacode-v0.1.zip",
    "publish": "true"
  }'
```