---
date: 2018-09-04
title: Specifying a subnet during Bake stage
categories:
   - AWS
description: Specifying a subnet during Bake stage
type: Document
---

## How To

There may be occassions where it's useful to specify the subnet used in your _Bake_ stage. Targeting a specific subnet can ensure your [Packer Builders](https://www.packer.io/docs/builders/index.html) (which Spinnaker relies on for baking images) are assigned a subnet which meets the requirements of the network (e.g. a subnet which does not auto-assign a public IP).

***

Specifying a subnet is handled through the _Extended Attributes_ of the _Bake_ stage's _Bake Configuration_. You will need to know the _id_ of your subnet and associate it with the _aws_subnet_id_ key.

![screenshot of bake stage](https://cl.ly/a79c60fd317f/%255B5fece2a398c53902605f183ca343e4e5%255D_subnet-in-bake-stage.png)

