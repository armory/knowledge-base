---
date: 2019-07-16
title: Pipelines-as-Code Workflow
categories:
   - concepts
description: How to incorporate Armory's Pipelines-as-Code feature into your teams' workflows
type: Document
---

### What is Pipelines-as-Code?

The Armory Pipelines-as-Code feature allows you to define your pipelines as
JSON templates within your code repositories.  This keeps the deployment
definition close to the code that is being deployed, and lets you revise
that definition with all the benefits of source control management.

---

> * How is this different from Managed Pipeline Templates?
> 
> The OSS Spinnaker community has been working on [Managed Pipeline
> Templates](https://blog.spinnaker.io/spinnaker-managed-pipeline-templates-v2-taking-shape-c7503d0a608d)
> which is a very similar concept, but Armory's Pipelines-as-Code feature
> takes this a step further by adding automation to the pipeline updates (see
> below)

---

### Intended Workflow

The Pipelines-as-Code feature is intended to make it much faster and easier
for developers to get a brand new application up and running.  The general
workflow for new projects is:

1. Developer creates a new project in source control
2. They create a Dinghyfile to build the application and pipelines in
   Spinnaker (even easier if there is a Module Repo set up with a
templatized set of pipelines)
3. When the code is committed to "master", Armory Spinnaker picks up the
   Dinghyfile, renders it, and applies it to Spinnaker, creating the
application and the pipelines.

Job done!  If everything's been configured properly, your developers should
be able to deploy their code using a previously-proven pipeline model
without ever having had to go into Spinnaker to configure anything.

As an added bonus, the pipeline definitions have now been saved in source
control, along with the rest of the project's files.  If changes are made
to the Dinghyfile, when committed/merged into the "master" branch, the
pipelines are automatically re-rendered and updated.






