---
date: 2018-07-09
title: Why hybrid cloud?
categories:
   - concepts
description: Reasons to use hybrid cloud
type: Document
---

We often receive a number of questions about reasons for deploying to hybrid cloud.  Here are some of the questions and answers:

*** 

Q: For a greenfield situation why would large enterprises deploy to multiple clouds rather than a single cloud?

A: Yes, for some.  This is because their customers are in those clouds.  Often, enterprises will have customers with customer data in multiple clouds; in these cases, it's often legally necessary to host or process their customer data in the clouds where their customers are operating.

Others have compliance concerns where they can only operate out of certain regions and thus will need to operate in those datacenters.  And some want particular functionality that other clouds don't have.  GCP's machine learning tools and APIs are a reason for that. 

Often EC2 (or any other VM) used in conjunction with k8s is indicative of a transition to a new tech, not a deliberate decision to simultaneously use both.

This isn't always the case, though.  In many cases organizations want to completely operate out of k8s, however in other cases there are some workloads that are not perceived ready to move to k8s or may never want to move.  Some organizations may have a set of applications that are well suited to Kubernetes and containers, and a different set of applications that work better with VMs (applications that may be resource intensive and require a strict, well-defined set of resources are a good example of this).

We are 100% confident in saying that true enterprise organizations are going to be split across multiple clouds (AWS/GCP/Azure) and multiple targets (VMs, k8s, functions) forever.   As GCP/AWS/Azure continue to innovate the benefits of using Spinnaker to manage multiple cloud targets, and multiple types of cloud targets, will continue to grow.