---
date: 2019-10-22
title: SSH Keys in Terraformer
categories:
   - Installation
description: This article show how to install SSH Keys into the Terraformer container.
type: Document
---

## Background

This document will show you how to install SSH Keys into the Terraformer container.  This process will require
modifications to the deployment of Terraformer running in Kubernetes.

This is due to this Kubernetes issue: https://github.com/kubernetes/kubernetes/issues/2630

This is based on the workaround from here: https://stackoverflow.com/a/57908921

## Prerequisites

  Terraformer installed in the Armory Spinnaker cluster.
  The SSH Key should already be created and added as a Deploy Key to the Git repository.

## Create the Secret

On local workstation, create a directory and place the SSH Key and any other required authentication information inside:

1. Create the directory:

  `mkdir ssh`

2. Copy the SSH Key:

  `cp $SSH_KEY_FILE ssh/id_rsa`

3. Copy any other authentication information that's needed:

  `cp $GOOGLE_APPLICATION_CREDENTIALS ssh/account.json`

4. Create a config file for SSH to ignore the known_hosts checks:

  `echo "StrictHostKeyChecking no" > ssh/config`

5. Create the secret using `kubectl`:

    `kubectl create secret generic spin-terraformer-sshkey -n spinnaker-system --from-file=id_rsa=ssh/id_rsa --from-file=config=ssh/config --from-file=account.json=ssh/account.json`

In this example, we create a secret with the SSH key, a config to ignore known_hosts file issues, and the GCP service
account information to access the backend bucket that Terraform is configured to use.

## Update the Manifest

Next, the K8s manifest needs to be updated to include a few things.  

1. First, update the secret and an empty directory volume
that will contain the copy of the secret with the correct uid and permissions:
    ```
    volumes:
    - name: spin-terraformer-sshkey
      secret:
        defaultMode: 420
        secretName: spin-terraformer-sshkey
    - name: ssh-key-tmp
      emptyDir:
        sizeLimit: "128k"
    ```

2. Second, define an init container that copies the secret contents to the empty directory and set the permissions and
ownership correctly.  The Spinnake user uses user id 1000:
    ```
    ### Adding to set the ownership of the ssh keys
    initContainers:
    - name: set-key-ownership
      image: alpine:3.6
      command: ["sh", "-c", "cp /key-secret/* /key-spin/ && chown -R 1000:1000 /key-spin/* && chmod 600 /key-spin/*"]
      volumeMounts:
      - mountPath: /key-spin
        name: ssh-key-tmp
      - mountPath: /key-secret
        name: spin-terraformer-sshkey
    ```

3. Mount the (not so) empty directory into the Terraformer container at the `/home/spinnaker/.ssh` location:
    ```
    volumeMounts:
    - mountPath: /home/spinnaker/.ssh
      name: ssh-key-tmp
    ```

Finally, add this to the envs to get the GCP service account to work for the S3 bucket.  This isn't necessary
for the SSH Keys, but completes the example:
```
- env:
  - name: GOOGLE_APPLICATION_CREDENTIALS
    value: /home/spinnaker/.ssh/account.json
```

## Result

After the modifications are in place, and Terraformer has time to redeploy via the replica set, Terraform should be able
to clone Git repositories via SSH as long as the repository has the SSH Key added as a deploy key.  Halyard shouldn't
overwrite these changes, but it would be good to back this up.


