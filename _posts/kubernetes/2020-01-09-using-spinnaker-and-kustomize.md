---
date: 2020-01-09
title: Deploying manifests using Kustomize from Spinnaker 2.17
description: Use Armory Spinnaker to deploy manifests produced by Kustomize
categories:
  - Kubernetes
type: Document
---

### Preconditions
Kustomize in 1.17+ requires the [git/repo](https://www.spinnaker.io/reference/artifacts/types/git-repo/) artifact type. 

## Enabling Kustomize

Kustomize can be enabled by a feature flag in Armory Spinnaker 2.16 and later.

For Halyard, add the following line to `~/.hal/{DEPLOYMENT_NAME}/profiles/settings-local.js`:

	window.spinnakerSettings.feature.kustomizeEnabled = true;

Apply your changes to your Spinnaker deployment:  `hal deploy apply`. Wait until the pods are in a RUNNING state before proceeding.

You can now use the *KUSTOMIZE* option on a _Bake (Manifest)_ stage.

![](/images/kustomize-enable.png)

> **Note:** Sometimes you will need to clear the cache in your browser in order to see the new *KUSTOMIZE* option available on a _Bake (Manifest)_ stage.

## Build the Pipeline

For this example, we are going to use this [kustomize public repository](https://github.com/kubernetes-sigs/kustomize), specifically the *helloWorld* example.
* * *
### Step 1 - Add an Expected Artifact

Add a **git/repo** Expected Artifact in the _Configuration_ section:

- **Account** (Required): The `git/repo` account to use. 
- **URL** (Required): The location of the Git repository.
- **Branch** (Optional): The branch of the repository you want to use. _Defaults to  `master`._ 
- **Subpath** (Optional): By clicking `Checkout subpath`, you can optionally pass in a relative subpath within the repository. This provides the option to checkout only a portion of the repository, thereby reducing the size of the generated artifact.

> **Note:** In order to execute the pipeline mannualy, it is necesary to check the *Use Default Artifact* and also fill the fields (same information above).

![](/images/kustomize-expected-artifact.png)

### Step 2 - Add a Bake (Manifest) stage

Add a **Bake (Manifest)** stage and choose the Render Engine *KUSTOMIZE*. Then, select the Expected Artifact you created in step 1 and specify the path for the **kustomization.yaml** file.

 ![](/images/kustomize-bake.png)

### Step 3 - Produce the Artifact

Spinnaker returns the _manifest_ in a Base64 encoded file, so it is necessary to Produce a single Base64 Artifact in this Bake (Manifest) stage:

![](/images/kustomize-base64.png)

### Step 4 - Deploy

Finally, add a **Deploy (Manifest)** stage. Make sure to select the _Manifest Source_: **Artifact** and select the Base64 Artifact produced by the _Bake (Manifest)_ stage.

![](/images/kustomize-deploy.png)

## Run the pipeline

After you execute the pipeline, you can see the manifest generated in YAML format by clicking on the _Baked Manifest YAML_ link: 

![](/images/kustomize-execution.png)

Also in the _Deploy_ stage you can see the Kubernetes objects as result of the manifest deployment.
