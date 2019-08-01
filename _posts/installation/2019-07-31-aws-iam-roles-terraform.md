---
date: 2019-07-31
title: AWS IAM roles from Terraform Script
categories:
   - Installation
description: This article show how to create the AWS IAM roles from a Terraform Script
type: Document
---

## Background

This document will show you how to create the AWS IAM roles from a Terraform Script.  This document is based on the idea to automatize this mannual process [Deploying to AWS from Spinnaker (using IAM instance roles)](https://docs.armory.io/spinnaker-install-admin-guides/add-aws-account-iam/) using a terraform script. That means you have two options to create the AWS IAM roles. 


## Preconditions

  Terraform installed.
  You should have a profile configured in your `~/.aws/credentials` file.

## Variables

In the file "terraform.tfvars" you need to set some variables in order to execute properly.

- provider-region = The region
- provider-profile = the name of the profile in ~/.aws/credentials
- node-name = Find one of the nodes which is part of your EKS or other Kubernetes cluster and get the name Role i.e "arn:aws:iam::0123456789:role/armory-spin-hal-aws-dev-node" and copy just the name `armory-spin-hal-aws-dev-node`
- managed-role-name = The name of the managed role, whatever you want "TestSpinnakerManagedRole"
- managing-policy = The name of the managed policy, whatever you want "TestDevSpinnakerManagingPolicy"
- pass-role-name = The name of the pass-role, whatever you want "TestPassRole"
- base-role-name = The name of the base-role, whatever you want "TestBaseIAMRole"
- role-policy = The name of the role-policy, whatever you want "TestEc2CloudForm"
- managing-role-instance-profile = the name of the Managing Role Instance Profile, whatever you want "TestInstanceProfile"

## How to Use

 1. Download the terraform script from: [https://github.com/armory/terraform.git](https://github.com/armory/terraform.git) and open the `aws-roles` folder

 2. Edit the terraform.tfvars and fill with the corresponding values.

 3. Run the follow commands.

```
terraform init 
terraform plan -var-file=terraform.tfvars
terraform apply -var-file=terraform.tfvars
```

## Result

After run the script the roles and policies should be created and the output of the script is the instructions that you will need to execute in order to enable the aws account in halyard. 

```
Outputs:

commands = Run this commands in order: 
export AWS_ACCOUNT_NAME=aws-dev-1 
export ACCOUNT_ID=569630529054 
export ROLE_NAME=role/SpinnakerManagedRoleTerraform 
hal config provider aws account add $AWS_ACCOUNT_NAME --account-id $ACCOUNT_ID --assume-role $ROLE_NAME 
hal config provider aws enable 
hal config provider aws account edit $AWS_ACCOUNT_NAME --regions us-east-1,us-west-2 
hal deploy apply
```


