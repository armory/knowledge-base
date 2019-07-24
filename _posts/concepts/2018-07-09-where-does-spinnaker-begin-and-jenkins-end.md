---
date: 2018-07-09
title: Where does Spinnaker begin and Jenkins end?
categories:
   - concepts
description: Spinnaker takes advantage of the existing Jenkins ecosystem and uses Jenkins behinds the scenes. Spinnaker was designed from the ground up to be a cloud deployment tool combining Continuous Integration and Continuous Delivery.
type: Document
---

### Question:

Where does Spinnaker begin and Jenkins end?

***

### Answer:

Spinnaker is not a build server. It was never designed that way. While in theory Spinnaker could implement its own service that acts as a build server, there is no need to do this. Spinnaker takes advantage of the existing Jenkins ecosystem and uses Jenkins behinds the scenes. Spinnaker was designed from the ground up to be a cloud deployment tool combining Continuous Integration and Continuous Delivery. This means that instead of just executing arbitrary tasks, it has first class support for cloud concepts.

***

### More Resources: 

- [How Spinnaker fits into the Continuous Delivery puzzle](https://blog.armory.io/how-spinnaker-fits-into-the-continuous-delivery-puzzle/)
- [Spinnaker is not a Build Server (and other misconceptions)](https://blog.armory.io/spinnaker-is-not-a-build-server-and-other-misconceptions/)
