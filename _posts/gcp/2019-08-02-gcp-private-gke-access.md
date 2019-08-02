---
date: 2019-08-02
title: Configuring private GKE cluster access for Kubectl    
categories:
   - GCP
description: A guide to how to configure access for Kubectl to private Google Kubernetes Engine cluster with different level of restricted access 
type: Document
---

## Objectives
1. How to update kubeconfig for Kubectl to access private GKE cluster using *gcloud* cli
2. How to update kubeconfig with GKE cluster master private endpoint  
3. How to authenticate Kubectl for GCP user and service account 

***
## Two endpoints in private GKE cluster

A private GKE cluster is an important means to isolate a cluster from public internet. Private cluster will enable its nodes to have internal RFC-1918 IP address only - **private nodes**. This is the characteristics definition of private cluster in GKE.

Further, **in a private cluster, the cluster master has two endpoints:**

**1) Private endpoint:** This is the internal IP address of the master, behind an internal load balancer in the master's VPC network. Nodes communicate with the master using the private endpoint. Any VM in your VPC network, and in the same region as your private cluster, can use the private endpoint.

**2) Public endpoint:** This is the external IP address of the master. You can configure access to the public endpoint. In the most restricted case, there is no access to the public endpoint. You can relax the restriction by authorizing certain address ranges to access the public endpoint. You can also remove all restriction and allow anyone to access the public endpoint.

## Master endpoint control configurations options

In a private cluster one can control access to the cluster master endpoints. Cluster master can have varying access configurations that can be tuned per security requirements as you will see below.
1. Public endpoint access *disabled*.
2. Public endpoint access *enabled* and master authorized networks access *enabled*
3. Public endpoint access *enabled* and master authorized networks access *disabled*.

### KB reference: 
- [Private GKE cluster] (https://kb.armory.io/gcp/gcp-private-gke-and-cluster-master-endpoints-access/)

## Cluster Access Overview

A kubernetes cluster uses a YAML file called *kubeconfig* to store cluster authentication information. This information will be used by kubectl tool to access cluster. By default, the file is saved locally at $HOME/.kube/config. 

When you create a cluster using GCP Console or using gcloud CLI from a different computer, your environment's *kubeconfig* is not updated. Additionally, if a project team member uses gcloud to create a cluster from their computer, their kubeconfig is updated but yours is not. 

If you want to update cluster authentication and want to use kubectl, follow the instruction below to refresh the cluster credentials in your local kubeconfig:
   ```
   gcloud container clusters get-credentials [your-clustert-name]
   ```

## Generating a kubeconfig using a private cluster's master internal IP address  

You can see that the above command gets the cluster endpoint as part of updating kubeconfig. By default, running *get-credentials* against a private GKE cluster sets the external IP address as the endpoint. 

If you want to change this default behavior and want to use the internal IP as the endpoint i.e. private endpoint, follow the instructions below to refresh the cluster credentials in your local kubeconfig:

   ```
   gcloud container clusters get-credentials --internal-ip [your-clustert-name]
   ```

By specifying the flag --internal-ip, one can write / replace a private cluster's internal IP address to kubeconfig. This enables access to private endpoint of GKE cluster with public endpoint access *disabled*.

## Authenticating Kubectl

When you use gcloud cli to set up your environment's kubeconfig for a new or existing cluster, gcloud gives kubectl the same credentials used by gcloud itself. For example, if you use gcloud auth login, your personal credentials are provided to kubectl, including the userinfo.email scope. This allows the GKE cluster to authenticate the kubectl client.

Alternatively, you may choose to configure kubectl to use the credentials of a GCP service account, while running on a Compute Engine instance. However, by default, the userinfo.email scope is not included in credentials created by Compute Engine instances. Therefore, you must add this scope explicitly, such as by using the --scopes flag when the Compute Engine instance is created.

Once users or GCP service accounts are authenticated, they must also be authorized to perform any action on a GKE cluster. See role-based access control can be used to configure authorization (not in scope of this article).

   ```
   kubectl create clusterrolebinding cluster-admin-binding \
     --clusterrole cluster-admin \
     --user [target user's / service account GCP login email id]
   ```

All GKE clusters are configured to accept GCP user and service account identities, by validating the credentials presented by kubectl and retrieving the email address associated with the user or service account identity. As a result, the credentials for those accounts must include the userinfo.email OAuth scope in order to successfully authenticate.

### More Resources: 
- [Private GKE cluster master endpoint restriction](https://kb.armory.io/gcp/gcp-private-gke-and-cluster-master-endpoints-access)
