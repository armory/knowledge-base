---
date: 2019-07-18
title: Understanding and creating private GKE cluster with restricted access to cluster master endpoints
categories:
   - GCP
description: A guide to create private Google Kubernetes Engine cluster with different level of restricted access to cluster master endpoints 
type: Document
---

### What you will learn?
1. What defines a private cluster in GKE
2. Notable features, restrictions of private cluster in GKE
3. How to create private kubernetes cluster in Google cloud (GCP) with different levels of configuration to control access to the cluster master public endpoint

***

### Objective of private cluster:

Overaching objective of a private cluster is to ensure workloads on Kubernetes are isolated from the public internet. While there are many ways to achieve this isolation, a private GKE cluster is an important means. Private cluster will enable its nodes to have internal RFC-1918 IP address only - private nodes. This is the characteristics definition of private GKE cluster.

However, in a private cluster one can control access to the cluster master. Cluster mater can have varying access configurations that can be tuned per security requirements as you will see below. 

### Characteristics of a private cluster:
1. Private cluster will have private nodes i.e. nodes with no external IP addresses. Private nodes do not have outbound internet access. Traffic between nodes and the master is routed entirely using internal IP addresses. 
2. A private cluster can use an HTTP(s) load balancer or a network loadbalancer to accept incoming traffic, even though nodes have no external IP addresses.
3. A private cluster can use an internal load balancer to accept traffic from within an VPC network.
4. A private cluster require VPC network peering. Why? While your VPC network contains the cluster nodes, a separate VPC network in a google owned project contains the master. 
5. As called out, private nodes do not have outbound access. *Private Google Access* provides private nodes and their workloads with limited outbound access to Google cloud platform APIs and services over Google's private network.Example, *Private Google Access* makes it possible for private nodes to pull container images from *GCR* and send logs to *Stackdriver*.
6. A private cluster must be a VPC native cluster which has Alias IP ranges enabled.

### Overview:
Every cluster has a Kubernetes API server called the master. Masters run on VMs in Google-owned projects. In a private cluster, you can only control access to the master. 

In a private cluster, the master has two endpoints:
**Private endpoint:** This is the internal IP address of the master, behind an internal load balancer in the master's VPC network. Nodes communicate with the master using the private endpoint. Any VM in your VPC network, and in the same region as your private cluster, can use the private endpoint.

**Public endpoint:** This is the external IP address of the master. You can configure access to the public endpoint. In the most restricted case, there is no access to the public endpoint. You can relax the restriction by authorizing certain address ranges to access the public endpoint. You can also remove all restriction and allow anyone to access the public endpoint.

### Master endpoints access congrol options:
GCP offers three configurations to control access to the GKE cluster master endpoints. 
1. Public endpoint access *disabled*.
2. Public endpoint access *enabled* and master authorized networks access *enabled*
3. Public endpoint access *enabled* and master authorized networks access *disabled*.

### Gcloud creation flags to create private cluster:
You'll create the private cluster by using the following *gcloud* cluster creation flags.

1. *Public endpoint access disabled*
--enable-ip-alias
--enable-private-nodes
--enable-private-endpoint
--enable-master-authorized-networks

2. *Public endpoint access enabled and master authorized networks access enabled*
--enable-ip-alias
--enable-private-nodes
--enable-master-authorized-networks

3. *Public endpoint access enabled and master authorized networks access disabled*.
--enable-ip-alias
--enable-private-nodes
--no-enable-master-authorized-networks

 
### Gcloud command examples to create private cluster:

1. *Public endpoint access disabled*

    ```
   gcloud container clusters create private-cluster-0 \
    --create-subnetwork name=my-subnet-0 \
    --enable-master-authorized-networks \
    --enable-ip-alias \
    --enable-private-nodes \
    --enable-private-endpoint \
    --master-ipv4-cidr 172.16.0.0/28 \
    --no-enable-basic-auth \
    --no-issue-client-certificate
    ...
    ```

where:
--enable-private-endpoint *indicates no client access to public endpoints.*
--enable-private-nodes *indicates that the cluster's nodes do not have external IP addresses.*
--enable-master-authorized-networks *specifies that access to the public endpoint is restricted to internal IP address ranges that you authorize.*
--create-subnetwork name=my-subnet-0 *causes GKE to automatically create a subnet named my-subnet-2.*
--enable-ip-alias *makes the cluster VPC-native.*

**Notes** You do not need to explicitly authorize the internal IP address ranges of nodes and Pods within my-subnet-0. Those internal addresses are always authorized to communicate with the private endpoint. External IP address are disabled. If you want to access the cluster master from outside my-subnet-0, you must authorize at least one internal IP address range to have access to the private endpoint as below.

    ```
gcloud container clusters update private-cluster-0 \
    --enable-master-authorized-networks \
    --master-authorized-networks 10.128.0.3/32
  ```


2. *Public endpoint access enabled and master authorized networks access enabled*

    ```
   gcloud container clusters create private-cluster-1 \
    --create-subnetwork name=my-subnet-1 \
    --enable-master-authorized-networks \
    --enable-ip-alias \
    --enable-private-nodes \
    --master-ipv4-cidr 172.16.0.16/28 \
    --no-enable-basic-auth \
    --no-issue-client-certificate
    ....
    ```

where:
--enable-private-nodes *indicates that the cluster's nodes do not have external IP addresses.*
--enable-master-authorized-networks *specifies that access to the public endpoint is restricted to IP address ranges that you authorize.*
--create-subnetwork name=my-subnet-1 *causes GKE to automatically create a subnet named my-subnet-1.*
--enable-ip-alias *makes the cluster VPC-native.*

**Notes** Use --master-authorized-networks to specify external and internal IP addresses that can access the master.

3. *Public endpoint access enabled and master authorized networks access disabled*.

    ```
   gcloud container clusters create private-cluster-2 \
    --create-subnetwork name=my-subnet-2 \
    --no-enable-master-authorized-networks \
    --enable-ip-alias \
    --enable-private-nodes \
    --master-ipv4-cidr 172.16.0.32/28 \
    --no-enable-basic-auth \
    --no-issue-client-certificate
    ....
    ```

where:
--enable-private-nodes *indicates that the cluster's nodes do not have external IP addresses.*
--no-enable-master-authorized-networks *specifies that access to the public endpoint is not restricted to IP address ranges that you authorize.*
--create-subnetwork name=my-subnet-2 *causes GKE to automatically create a subnet named my-subnet-2.*
--enable-ip-alias *makes the cluster VPC-native.*

**Notes** If you enable access to the master's public endpoint without enabling master authorized networks, access to the master's public endpoint is not restricted.

### Accessing private cluster using Kubectl

1. **Public endpoint access disabled**
*From private nodes:* Always uses the private endpoint. kubectl must be configured to use the private endpoint. 

*From other VMs in the cluster's VPC network:* Other VMs can use kubectl to communicate with the private endpoint only if they are in the same region as the cluster and their internal IP addresses are included in the list of master authorized networks.

*From public IP addresses:* Never.

2. **Public endpoint access enabled and master authorized networks access enabled**
*From private nodes:* Always uses the private endpoint. kubectl must be configured to use the private endpoint. 

*From other VMs in the cluster's VPC network:* Other VMs can use kubectl to communicate with the private endpoint only if they are in the same region as the cluster and their internal IP addresses are included in the list of master authorized networks.

*From public IP addresses:* Machines with public IP addresses can use kubectl to communicate with the public endpoint only if their public IP addresses are included in the list of master authorized networks. This includes machines outside of GCP and GCP VMs with external IP addresses.

3. **Public endpoint access enabled and master authorized networks access disabled**
*From private nodes:* Always uses the private endpoint. kubectl must be configured to use the private endpoint. 

*From other VMs in the cluster's VPC network:* Other VMs can use kubectl to communicate with the private endpoint only if they are in the same region as the cluster.

*From public IP addresses:* Any machine with a public IP address can use kubectl to communicate with the public endpoint. This includes machines outside of GCP and GCP VMs with external IP addresses.

### Clean-up

1. **Delete the private cluster using Gcloud**
    ```
gcloud container clusters delete -q private-cluster-0 private-cluster-1
    ```

2. **Delete the network using Gcloud**
    ```
    gcloud compute networks delete net-0
   ```

### More Resources: 
- [Completely private GKE cluster](https://medium.com/google-cloud/completely-private-gke-clusters-with-no-internet-connectivity-945fffae1ccd)
