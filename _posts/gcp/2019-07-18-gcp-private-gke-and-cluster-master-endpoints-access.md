---
date: 2019-07-18
title: Private GKE with restricted access to cluster master endpoints
categories:
   - GCP
description: A guide to private Google Kubernetes Engine cluster with different level of restricted access to cluster master endpoints 
type: Document
---

## Objectives
1. Reason for choosing private GKE cluster 
2. Notable features, restrictions of private GKE cluster  
3. How to create private GKE cluster in Google cloud (GCP) with different levels of configuration to control access to the cluster master public endpoint

***
## Why private cluster

Overaching objective of a private cluster is to ensure workloads on Kubernetes are isolated from the public internet. While there are many ways to achieve this isolation, a private GKE cluster is an important means. Private cluster will enable its nodes to have internal RFC-1918 IP address only - **private nodes**. This is the characteristics definition of private cluster in GKE.


However, **in a private cluster one can control access to the cluster master**. Cluster master can have varying access configurations that can be tuned per security requirements as you will see below. 

## Characteristics 
1. Private cluster will have private nodes i.e. node with no external IP address. Private nodes do not have outbound internet access. Traffic between nodes and the master is routed entirely using internal IP addresses. 
2. A private cluster can use an HTTP(s) load balancer or a network loadbalancer to accept incoming traffic, even though nodes have no external IP addresses.
3. A private cluster can use an internal load balancer to accept traffic from within an VPC network.
4. A private cluster require VPC network peering. Why? While your VPC network contains the cluster nodes, a separate VPC network in a google owned project contains the master. 
5. As called out, private nodes do not have outbound access. *Private Google Access* provides private nodes and their workloads with limited outbound access to Google cloud platform APIs and services over Google's private network. For example, *Private Google Access* makes it possible for private nodes to pull container images from *GCR* and send logs to *Stackdriver*.
6. A private cluster must be a VPC native cluster which has Alias IP ranges enabled.

## Overview
Every cluster has a Kubernetes API server called the master. Master run on VMs in Google-owned project. In a private cluster, you can only control access to the master. In GKE, the cluster master has two endpoints:

**1) Private endpoint:** This is the internal IP address of the master, behind an internal load balancer in the master's VPC network. Nodes communicate with the master using the private endpoint. Any VM in your VPC network, and in the same region as your private cluster, can use the private endpoint.

**2) Public endpoint:** This is the external IP address of the master. You can configure access to the public endpoint. In the most restricted case, there is no access to the public endpoint. You can relax the restriction by authorizing certain address ranges to access the public endpoint. You can also remove all restriction and allow anyone to access the public endpoint.

## Control options 

GKE offers three configurations to control access to the private cluster master endpoints. 
1. Public endpoint access *disabled*.
2. Public endpoint access *enabled* and master authorized networks access *enabled*
3. Public endpoint access *enabled* and master authorized networks access *disabled*.


Below cluster creation flags need to be used with **gcloud** command line tool:

1. **Public endpoint access *disabled***
   ```
   --enable-ip-alias
   --enable-private-nodes
   --enable-private-endpoint
   --enable-master-authorized-networks
   ```

2. **Public endpoint access *enabled* and master authorized networks access *enabled***
   ```
   --enable-ip-alias
   --enable-private-nodes
   --enable-master-authorized-networks
   ```

3. **Public endpoint access *enabled* and master authorized networks access *disabled***
   ```
   --enable-ip-alias
   --enable-private-nodes
   --no-enable-master-authorized-networks
   ```
 
## Create private cluster

1. **Public endpoint access *disabled***

   ```
   gcloud container clusters create private-cluster-0 \
    --create-subnetwork name=my-subnet-0 \   ** causes GKE to automatically create a subnet named my-subnet-0
    --enable-master-authorized-networks \    ** access to the public endpoint is restricted to internal IP address ranges that you authorize
    --enable-ip-alias \                      ** makes the cluster VPC-native
    --enable-private-nodes \                 ** indicates that the cluster's nodes do not have external IP addresses
    --enable-private-endpoint \              ** indicates no client access to public endpoints
    --master-ipv4-cidr 172.16.0.0/28 \
    --no-enable-basic-auth \
    --no-issue-client-certificate
    ...
   ```
**Note** You do not need to explicitly authorize the internal IP address ranges of nodes and pods within *my-subnet-0*. Those internal addresses are always authorized to communicate with the master private endpoint. Internal IP address range outside the subnet require explicit network authorization. External endpoint of master is *disabled*. This configuration enables more secure private cluster.

2. **Public endpoint access *enabled* and master authorized networks access *enabled***

   ```
   gcloud container clusters create private-cluster-1 \
    --create-subnetwork name=my-subnet-1 \   ** causes GKE to automatically create a subnet named my-subnet-1
    --enable-master-authorized-networks \    ** access to the public endpoint is restricted to IP address ranges that you authorize
    --enable-ip-alias \                      ** makes the cluster VPC-native
    --enable-private-nodes \                 ** indicates that the cluster's nodes do not have external IP addresses
    --master-ipv4-cidr 172.16.0.16/28 \
    --no-enable-basic-auth \
    --no-issue-client-certificate
    ....
   ```
**Note** If you enable access to the master's public endpoint without enabling master authorized networks, access to the master's public endpoint is not restricted. This is not a recommended for security reason. Explicit network authorization against master public endpoint is required.

3. **Public endpoint access *enabled* and master authorized networks access *disabled***

   ```
   gcloud container clusters create private-cluster-2 \
    --create-subnetwork name=my-subnet-2 \   ** causes GKE to automatically create a subnet named my-subnet-2
    --no-enable-master-authorized-networks \ ** access to the public endpoint is not restricted by GKE service
    --enable-ip-alias \                      ** makes the cluster VPC-native
    --enable-private-nodes \                 ** indicates that the cluster's nodes do not have external IP addresses
    --master-ipv4-cidr 172.16.0.32/28 \  
    --no-enable-basic-auth \ 
    --no-issue-client-certificate
    ....
   ```
**Note** Above configuration is not a recommended for security reason. It allows access to cluster master public endpoint. And relies on VPC firewalls to restrict access to cluster master. 

**Additional note** You can use *--master-authorized-networks* flag to specify external and internal IP addresses that can access the master as given below.

   ```
   gcloud container clusters update private-cluster-0 \
    --enable-master-authorized-networks \
    --master-authorized-networks 10.128.0.3/32  ** internal / external IP can be specified
   ```

## Clean-up

1. **Delete the private cluster using Gcloud**
   ```
   gcloud container clusters delete -q private-cluster-0 private-cluster-1 private-cluster-2
   ```

2. **Delete the network using Gcloud**
   ```
   gcloud compute networks delete my-net-0   ** deleting network will delete its subnets as well 
   ```

### More Resources: 
- [Completely private GKE cluster](https://medium.com/google-cloud/completely-private-gke-clusters-with-no-internet-connectivity-945fffae1ccd)
