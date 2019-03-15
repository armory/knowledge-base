---
date: 2019-03-13
title: Upgrading from OSS to Armory Spinnaker
categories:
   - Installation
description: Upgrading from OSS to Armory Spinnaker
type: Document
---

If you are currently on OSS Spinnaker and interested in upgrading to Armory Spinnaker, you can easily upgrade if you used Halyard to install your Spinnaker cluster (you can also easily downgrade back to OSS Spinnaker)

At a high level, the process to do this is as follows:
* Back up your existing Halyard configuration
* Switch from OSS Spinnaker to Armory Spinnaker
* Update the Halconfig Spinnaker version from the OSS Spinnaker version to Armory Spinnaker
* Deploy Armory Spinnaker, and remove any conflicting fields from your Halconfig

Here are the actual steps:
## Back up your existing Halyard configuration
* From your existing Halyard, run `hal backup create`.  This will create a `tar` file; make sure you store this somewhere safe and external to your Halyard machine.

## Switch from OSS Spinnaker Halyard to Armory Spinnaker Halyard

* If you are using the OSS Halyard Docker image, run with something like this:

    ```bash
    docker run --name armory-halyard --rm \
        -v ${PWD}/.hal:/home/spinnaker/.hal \
        -v ${PWD}/.kube:/home/spinnaker/.kube \
        -v ${PWD}/.secret:/home/spinnaker/.secret \
        -it gcr.io/spinnaker-marketplace/halyard:stable
    ```

Then you can switch to Armory Halyard by just switching out the Docker image.  For example, as of 3/15/2019, the latest image is `docker.io/armory/halyard-armory:1.3.2` (you can recent/current image tags at https://hub.docker.com/r/armory/halyard-armory/tags).  Switch to the new image with this:

    ```bash
    docker run --name armory-halyard --rm \
        -v ${PWD}/.hal:/home/spinnaker/.hal \
        -v ${PWD}/.kube:/home/spinnaker/.kube \
        -v ${PWD}/.secret:/home/spinnaker/.secret \
        -it docker.io/armory/halyard-armory:1.3.2
    ```

* If you have Halyard directly installed on your machine, you can update to Armory Halyard with the Armory installer:

    ```bash
    curl -LO https://get.armory.io/halyard/install/latest/macos/InstallArmoryHalyard.sh 

    sudo bash InstallArmoryHalyard.sh --version 1.3.2
    ```

## Update the Halconfig Spinnaker version from the OSS Spinnaker version to Armory Spinnaker

In Halyard, run this command to get the latest version(s) of Armory Spinnaker:

    ```bash
    hal version list
    ```

Then, update your Halconfig to use the Armory Spinnaker version.  For example, if you're trying to use 2.2.0, run this:

    ```bash
    hal config version edit --version 2.2.0
    ```

## Deploy Armory Spinnaker, and remove any conflicting fields from your Halconfig

Because Armory Spinnaker is much more heavily tested than OSS Spinnaker, certain edge capabilities in OSS Spinnaker may not be available in Armory Spinnaker.

You can identify these by trying to apply your change:

```bash
$ hal deploy apply
- Get current deployment
  Failure
Problems in Global:
! ERROR Could not translate your halconfig: Unrecognized field
  "nodeSelectors" (class
  com.netflix.spinnaker.halyard.config.model.v1.node.DeploymentEnvironment), not
  marked as ignorable (14 known properties: "size", "initContainers",
  "updateVersions", "consul", "customSizing", "vault", "gitConfig", "location",
  "sidecars", "haServices", "accountName", "type", "hostAliases",
  "bootstrapOnly"])
at [Source: N/A; line: -1, column: -1] (through reference chain:
  io.armory.halyard.config.model.v1.node.ArmoryHalconfig["deploymentConfigurations"]->java.util.ArrayList[0]->com.netflix.spinnaker.halyard.config.model.v1.node.ArmoryDeploymentConfiguration["deploymentEnvironment"]->com.netflix.spinnaker.halyard.config.model.v1.node.DeploymentEnvironment["nodeSelectors"])

- Failed to get deployment name.
```

In this situation, go into your `~/.hal/config` file and remove the unrecognized field.  For example, for the above, remove `deploymentEnvironment.nodeSelectors` by commenting it out:

```yml
  deploymentEnvironment:
    size: SMALL
    type: Distributed
    accountName: spinnaker
    updateVersions: true
    consul:
      enabled: false
    vault:
      enabled: false
    location: jlee-oss-edge
    customSizing: {}
    sidecars: {}
    initContainers: {}
    hostAliases: {}
    # nodeSelectors: {}
    gitConfig:
      upstreamUser: spinnaker
    haServices:
      clouddriver:
        enabled: false
        disableClouddriverRoDeck: false
      echo:
        enabled: false
```

Then, repeat the `hal deploy apply` until it actually deploys (there are usually about 4-5 fields that have to be commented out.)

## Revert
When you want to go back to OSS Spinnaker, you can basically repeat the same process as above, with OSS Halyard.