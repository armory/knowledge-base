---
date: 2018-12-14
title: Spinnaker Short Demo
categories:
   - Armory Platform
description: Spinnaker Short Demo
type: Document
---

![Demo GIF](https://d2ddoduugvun08.cloudfront.net/items/1I1v1z1V131s072b2f1M/%5B44726d704afab65a42b0af1b0640a8f3%5D_Armory-Kubecon-Demo.gif)

This is a quick 1-minute GIF showing a simplified version of the pipeline execution on display at the Armory booth at Kubecon.

In it, we see the components of the pipeline definition:
* A configuration tab, on which we can configure settings, triggers, and notifications for the pipeline as a whole
* A *Kubernetes v2 Manifest* deployment stage to a "Dev" environment
* A *Manual Judgment* stage, which will wait for human intervention
* A *Kubernetes v2 Manifest* deployment stage to a "Prod" environment

We also trigger the pipeline, and watch as the dev environment gets updated from the old configuration (`dev: Hello Kubecon!`) to a new configuration (`dev: Hello Seattle!`).  Once an operator has validated that the dev environment is operating as expected, he can go in and approve the promotion to production.

Please feel free to reach out to use if you'd like to learn more at [hello@armory.io](hello@armory.io)!
