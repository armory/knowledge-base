---
date: 2020-01-09
title: Deploying manifests using Kustomize from Spinnaker 1.17
description: Use Armory Spinnaker to deploy manifests produced by Kustomize
categories:
  - Kubernetes
type: Document
---

### Preconditions
Kustomize in 1.17+ requires the [git/repo](https://www.spinnaker.io/reference/artifacts/types/git-repo/) artifact type. 

## Enabling Kustomize

Kustomize can be enabled by a feature flag in 1.16 and 1.17.

For Halyard, add the following line to `~/.hal/{DEPLOYMENT_NAME}/profiles/settings-local.js`

	window.spinnakerSettings.feature.kustomizeEnabled = true;


Once the file is created then perform a `hal deploy apply` and wait until the pods are RUNNING.

Doing this now you will be able to see the *KUSTOMIZE* option on the _Bake (Manifest)_ stage.

![](/images/kustomize-enable.png)

> **Note:** Sometimes you will need to clear cache in your browser in order to see the new *KUSTOMIZE* option available on the _Bake (Manifest)_ stage.

## Build the Pipeline

For this example we are going to use this [kustomize public repository](https://github.com/kubernetes-sigs/kustomize) specifically the *helloWorld* example.
* * *
### Step 1 - Add an Expected Artifact

The first step is to add a **git/repo** Expected Artifact in the _Configuration_ section as follows:

- **Account** (Required): The `git/repo` account to use. 
- **URL** (Required): The location of the Git repository.
- **Branch** (Optional): The branch of the repository you want to use. _[Defaults to  `master`]_ 
- **Subpath** (Optional): By clicking `Checkout subpath`, you can optionally pass in a relative subpath within the repository. This provides the option to checkout only a portion of the repository, thereby reducing the size of the generated artifact.

> **Note:** In order to execute the pipeline mannualy is necesary to check the *Use Default Artifact* and also fill the fields (same information avobe).

![](/images/kustomize-expected-artifact.png)

### Step 2 - Add a Bake (Manifest) stage

Second step is to add a **Bake (Manifest)** stage choosing the Render Engine *KUSTOMIZE*, select the Expected Artifact previously created and specify the path for the **kustomization.yaml** file.

 ![](/images/kustomize-bake.png)

### Step 3 - Produces Artifact

Spinnaker is going to return the _manifest_ in a Base64 encoded file, so is necessary to Produce a single Base64 Artifact in this Bake (Manifest) stage as follows:

![](/images/kustomize-base64.png)

### Step 4 - Deploy

Finally add a **Deploy (Manifest)** stage, just make sure to select the _Manifest Source_: **Artifact** and select the Base64 Artifact produced by the _Bake (Manifest)_ stage.

![](/images/kustomize-deploy.png)

## Run the pipeline

After execute the pipeline you can see the manifest generated in YAML format clicking on the _Baked Manifest YAML_ link: 

![](/images/kustomize-execution.png)

Also in the _Deploy_ stage you can see the Kubernetes objects as result of the manifest deployment.
