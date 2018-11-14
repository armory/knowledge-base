---
date: 2018-07-10
title: Multi-Target and Multi-Cloud Deployment Use Cases
categories:
   - Armory Platform
description: Armory Multi Cloud
type: Document
---

### We were recently asked: 

> "Can you share with me what you've seen as for actual multi-cloud usage from Armory customers?

> I know it's gaining popularity as a criteria, however I'm curious how many companies have actually landed with production workloads running across multiple public clouds. Also, is this typically closer to unrelated departments in organizations running on different clouds, or one team actually deploying to multiple clouds (e.g. one app running simultaneously on GCP/AWS)?"
	
### Our Answer:

Yes, we've found that almost all large customers (Global 2,000 & up) want to have a single pane of glass (like [Armory Spinnaker](https://www.armory.io/installed-spinnaker)) across the entire org for deploying workloads to multiple targets & clouds (not just in different parts of the business).
<br><br>
**Here's a sampling of deployment targets across several current Armory customers** using our Enterprise Spinnaker platform:

- **Global 2,000 public software company:** EC2, Kubernetes on-prem and in cloud
- **High growth public software company:** EC2, OpenStack, Kubernetes on-prem/cloud. (They also want to do GCP/GKE and Azure in the near future.)  
- **Private Global 2,000-sized software company:** EC2, Kubernetes on-prem/cloud, Azure.
- **Fortune 20 Public company:** EC2, Kubernetes on-prem/cloud, Mesos and their own private cloud
- **Fortune 20 Public company:** Kubernetes on-prem/cloud, EC2, PCF
- **Private high growth software company:** EC2, Kubernetes, GKE

***

### More Resources: 
- [Forbes: Armory Bets Big On Spinnaker - The Open Source Continuous Delivery Platform From Netflix](https://www.forbes.com/sites/janakirammsv/2018/08/23/armory-bets-big-on-spinnaker-the-open-source-continuous-delivery-platform-from-netflix/#4bb6e33372cd)
- [Multi Cloud Deployments with Spinnaker](https://blog.armory.io/multi-cloud-deployments-with-spinnaker/)
- [The Benefits of Multi-Cloud Deployments](https://blog.armory.io/the-benefits-of-multi-cloud-deployments/	)
