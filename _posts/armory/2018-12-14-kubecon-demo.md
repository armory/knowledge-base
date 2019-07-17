---
date: 2018-12-14
title: Spinnaker Short Demo
categories:
   - Armory Platform
description: Spinnaker Short Demo
type: Document
---

![Demo GIF](https://d2ddoduugvun08.cloudfront.net/items/1p2q253u0q3h3A2L2v35/Armory-Kubecon-Demo-Small.gif)

This is a quick 1-minute GIF showing a simplified version of the pipeline execution on display at the Armory booth at Kubecon.

You can find a narrated version of this [youtube link](https://www.youtube.com/watch?v=J45FYB7EjUY)

In it, we see the components of the pipeline definition:
* A configuration tab, on which we can configure settings, triggers, and notifications for the pipeline as a whole
* A *Kubernetes v2 Manifest* deployment stage to a "Dev" environment
* A *Manual Judgment* stage, which will wait for human intervention
* A *Kubernetes v2 Manifest* deployment stage to a "Prod" environment

We also trigger the pipeline, and watch as the dev environment gets updated from the old configuration (`dev: Hello Kubecon!`) to a new configuration (`dev: Hello Seattle!`).  Once an operator has validated that the dev environment is operating as expected, they can go in and approve the manual judgment stage, which starts the promotion to production.

If you'd like to learn more about Spinnaker, or how Armory can help you run Spinnaker at Enterprise Scale, reach out to us at [hello@armory.io](hello@armory.io)!
