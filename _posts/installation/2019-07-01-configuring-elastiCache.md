---
date: 2018-12-26
title: Configuring ElastiCache with Spinnaker
categories:
   - Installation
description: Configuring ElastiCache with Spinnaker
type: Document
---

## Why Configure ElastiCache with spinnaker

Spinnaker stores in redis it's pipeline execution history. The default configuration deploys a pod in the kubernetes cluster. If the pods gets restarted, all of the history gets lost.

Using ElastiCache removes this possibility.


## Configuring ElastiCache with Spinnaker

We are going to use a terraform script to create all that we need for this.
The script is located in this repository:
https://github.com/armory/terraform/tree/master/elasticache

This script creates an ElastiCache cluster using a Redis engine, it's based on an already existing vpc using its subnet ids, this to allow connectivity to the Spinnaker cluster.


## Variables

In the file "terraform.tfvars" you need to set some variables.

```
-subnet-1-cidr: the eks subnet cluster
-subnet-2-cidr = the eks subnet cluster
-availability_zones: zones where a node of the cluster will be available. 
This needs to be equal or less than the number of nodes in the cluster
-security_group_ids: The eks nodes sg
```

## How to Use

    terraform init 
    terraform plan -var-file=terraform.tfvars
    terraform apply -var-file=terraform.tfvars


## Result

After run the script the infrastructure should be created and the output of the script is the primary endpoint of the ElastiCache cluster.


## Validate spinnaker has connection to elasticache
To validate Spinnaker has connection to elasticache you need to exec into gate pod and run
`nc -zv <endpoint from output (elasticache primary endpoint in the aws console)> 6379`

you should see something like:

```
bash-4.4$ nc -zv myelasticache.usw2.cache.amazonaws.com 6379
myelasticache.usw2.cache.amazonaws.com (0.0.0.0:6379) open
```

If it's not open, you should verify the subnet ids, and security groups to validate it is correct.


## Configuring spinnaker with the cluster
To configure spinnaker to use this cluster follow:

Using elasticache in aws eks cluster:
Run terraformer scripts:  Need to output the dns endpoint

If terraformer is enabled:  add to terraformer-local.yml
```
      redis:
        host: <elasticache-endpoint>
      port: <port>
```

If dinghy is enabled:  add to dinghy-local.yml

```
  services:
    redis:
        address: <elasticache-endpoint>
```

On redis.yml

```
overrideBaseUrl: redis://<elasticache-endpoint>:<port>
skipLifeCycleManagement: true
```


Disable automatic Redis configuration in Gate by adding the following to your gate-local.yml file:
```
   redis:
     configuration:
       secure: true
```

To delete the default redis pod run:
`kubectl -n spinnaker delete deployment spin-redis`


