---
date: 2018-12-26
title: Installing OSS Spinnaker in Kubernetes
categories:
   - Installation
description: Installing OSS Spinnaker in Kubernetes
type: Document
---

_Note: If you're using Armory Spinnaker, a lot of the items in this document are handled automatically for you._

## Installing Spinnaker

This document will guide you through the initial installation of Open Source Spinnaker, in the following environment:

* Spinnaker installed in Kubernetes
* Using an S3 bucket as the configuration store
* Installed via Halyard, run as a Docker container

We'll do this by starting a Halyard docker container (with a container name of `oss-halyard`), and then interacting with the Halyard daemon in that container from another shell session.

Later articles may cover the following:

* Configuring Spinnaker to use an external Redis (rather than the default redis Kubernetes container)
* Configuring high availability
* Adding a Kubernetes account ([document here](2018-12-26-spinnaker-add-kubernetes.md))
* Adding an AWS account



## Start the Halyard container

Halyard can be run directly on a Linux machine or on OSX, but is somewhat more portable in a Docker container.  In order to to this, you should directly mount three directories into your docker container. It's useful to have these directories accessible from a shared root.  We also recommend adding an additional directory to store sensitive information passed to Spinnaker (Spinnaker secret management forthcoming):

* `.hal` (where all Halyard-specific configurations are stored)
* `.aws` (where AWS credentials, specifically for S3, are stored)
* `.kube` (where kubeconfigs are stored)
* `.secret` (where other sensitive information is stored)

Your AWS and Kubernetes credentials can be copied from your home directory dot directories to your current working directory, as necessary

```bash
# Copy dot directories to current directory
cp -r ~/.aws .
cp -r ~/.kube .
```


Run the container:
```bash
# Start the Docker container (without port forwarding)
docker run --name oss-halyard -it --rm \
  -v ${PWD}/.hal:/home/spinnaker/.hal \
  -v ${PWD}/.kube:/home/spinnaker/.kube \
  -v ${PWD}/.aws:/home/spinnaker/.aws \
  -v ${PWD}/.secret:/home/spinnaker/.secret \
  gcr.io/spinnaker-marketplace/halyard:stable
```


Then, in a separate terminal/shell session, enter the docker container with this:

```bash
docker exec -it oss-halyard bash

# Also, once in the container, you can run these commands for a friendlier environment to:
# - prompt with information
# - alias for ls
# - cd to the home directory
export PS1="\h:\w \u\$ "
alias ll='ls -alh'
cd ~
```

## Create a service account with which to install Spinnaker

Halyard will install Spinnaker in your Kubernetes cluster; it is a good idea to have a dedicated service account for Halyard to use to interact with your Kubernetes cluster.  First, we'll create the namespace where Spinnaker will live, and then create a service account and permissions to install Spinnaker into that namespace.

### Set up bash parameters in the container

First, we'll set up bash environment variables that will be used by later commands

*Do this in the Halyard container*

```bash

# Specify the name of the kubernetes context that has permissions to create the service account in your
# target cluster and namespace.  To get the list of contexts, you can run "kubectl config get-contexts"
export CONTEXT="aws-armory-dev"

# Enter the namespace that you want to install Spinnaker in.  This can already exist, or can be created.
export NAMESPACE="spinnaker-installation"

# Enter the name of the service account you want to create.  This will be created in the target namespace
export SERVICE_ACCOUNT_NAME="spinnaker"

# Enter the name of the role you want to create.  This will be created in the target namespace
export ROLE_NAME="spinnaker-role"

# Enter the name you want Spinnaker to use to identify the cloud provider account
export ACCOUNT_NAME="spinnaker"
```

### Create the namespace to install Spinnaker in

If the namespace does not exist, you can create it.

*Do this in the Halyard container*

```bash
kubectl --context ${CONTEXT} create ns ${NAMESPACE}
```

### Create the service account and credentials

Create a Kubernetes manifest that contains the service account, role, and rolebinding.
*Note that this may not have all necessary permissions, depending on what you're deploying - feel free to add additional permissions to this account as neccessary*

*Do this in the Halyard container*

```bash
# Create the file
tee ${NAMESPACE}-service-account.yml <<-'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: SERVICE_ACCOUNT_NAME
  namespace: NAMESPACE
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ROLE_NAME
  namespace: NAMESPACE
rules:
- apiGroups: [""]
  resources: ["namespaces", "events", "replicationcontrollers", "serviceaccounts", "pods/log"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods", "services", "secrets", "configmaps"]
  verbs: ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"]
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["list", "get"]
- apiGroups: ["apps"]
  resources: ["controllerrevisions", "statefulsets"]
  verbs: ["list"]
- apiGroups: ["extensions", "apps"]
  resources: ["deployments", "replicasets", "ingresses"]
  verbs: ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"]
# These permissions are necessary for Halyard to operate. We use this role also to deploy Spinnaker itself.
- apiGroups: [""]
  resources: ["services/proxy", "pods/portforward"]
  verbs: ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ROLE_NAME-binding
  namespace: NAMESPACE
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ROLE_NAME
subjects:
- namespace: NAMESPACE
  kind: ServiceAccount
  name: SERVICE_ACCOUNT_NAME
EOF

# Do string substitution with our bash variables
sed -i.bak \
  -e "s/NAMESPACE/${NAMESPACE}/g" \
  -e "s/SERVICE_ACCOUNT_NAME/${SERVICE_ACCOUNT_NAME}/g" \
  -e "s/ROLE_NAME/${ROLE_NAME}/g" \
  ${NAMESPACE}-service-account.yml

kubectl --context ${CONTEXT} apply -f ${NAMESPACE}-service-account.yml
```

### Create a minified kubeconfig

In order for Spinnaker to talk to a Kubernetes cluster, it must be provided a kubeconfig.  We're going to create a trimmed-down kubeconfig that only has the service account and token.

*Do this in the Halyard container*

```bash
#################### Create minified kubeconfig
NEW_CONTEXT=${NAMESPACE}-sa
KUBECONFIG_FILE="kubeconfig-${NAMESPACE}-sa"

SECRET_NAME=$(kubectl get serviceaccount ${SERVICE_ACCOUNT_NAME} \
  --context ${CONTEXT} \
  --namespace ${NAMESPACE} \
  -o jsonpath='{.secrets[0].name}')
TOKEN_DATA=$(kubectl get secret ${SECRET_NAME} \
  --context ${CONTEXT} \
  --namespace ${NAMESPACE} \
  -o jsonpath='{.data.token}')

TOKEN=$(echo ${TOKEN_DATA} | base64 -d)

# Create dedicated kubeconfig
# Create a full copy
kubectl config view --raw > ${KUBECONFIG_FILE}.full.tmp
# Switch working context to correct context
kubectl --kubeconfig ${KUBECONFIG_FILE}.full.tmp config use-context ${CONTEXT}
# Minify
kubectl --kubeconfig ${KUBECONFIG_FILE}.full.tmp \
  config view --flatten --minify > ${KUBECONFIG_FILE}.tmp
# Rename context
kubectl config --kubeconfig ${KUBECONFIG_FILE}.tmp \
  rename-context ${CONTEXT} ${NEW_CONTEXT}
# Create token user
kubectl config --kubeconfig ${KUBECONFIG_FILE}.tmp \
  set-credentials ${CONTEXT}-${NAMESPACE}-token-user \
  --token ${TOKEN}
# Set context to use token user
kubectl config --kubeconfig ${KUBECONFIG_FILE}.tmp \
  set-context ${NEW_CONTEXT} --user ${CONTEXT}-${NAMESPACE}-token-user
# Set context to correct namespace
kubectl config --kubeconfig ${KUBECONFIG_FILE}.tmp \
  set-context ${NEW_CONTEXT} --namespace ${NAMESPACE}
# Flatten/minify kubeconfig
kubectl config --kubeconfig ${KUBECONFIG_FILE}.tmp \
  view --flatten --minify > ${KUBECONFIG_FILE}
# Remove tmp
rm ${KUBECONFIG_FILE}.full.tmp
rm ${KUBECONFIG_FILE}.tmp
```

### Add the kubeconfig and cloud provider to Spinnaker
You should copy the kubeconfig to a place accessible to Halyard; this choice is left to the reader, but one option is `~/.secret/`, which should be mounted into your Halyard container.

*Do this in the Halyard container*

```bash
# Feel free to reference a different location
KUBECONFIG_DIRECTORY=~/.secret/
cp ${KUBECONFIG_FILE} ${KUBECONFIG_DIRECTORY}
export KUBECONFIG_FULL=$(realpath ${KUBECONFIG_DIRECTORY}${KUBECONFIG_FILE})

# Enable the Kubernetes cloud provider
hal config provider kubernetes enable
# Enable artifacts
hal config features edit --artifacts true

# Add account
hal config provider kubernetes account add ${ACCOUNT_NAME} \
  --provider-version v2 \
  --kubeconfig-file ${KUBECONFIG_FULL} \
  --namespaces ${NAMESPACE}
```

## Configure Halyard to install Spinnaker in Kubernetes

We're going to configure Halyard to install Spinnaker as distributed microservices, using the kubernetes Spinnaker account created in the previous step, in the correct namespace.

```bash
hal config deploy edit \
  --type distributed \
  --account-name ${ACCOUNT_NAME} \
  --location ${NAMESPACE}
```

## Configure Halyard to use S3 for Spinnaker's persistent storage

Spinnaker needs a place to store persistent configurations.  This can be in an S3 bucket or in an S3-compatible repoistory.  For now, we're assuming you have / can create the following:
* An S3 bucket
* An IAM account that has full permissions on the S3 bucket
* A set of AWS creds for the IAM account

### Creating the bucket and creds (if you don't have them already):

#### Create the bucket

1. Log into your AWS account, and navigate to S3
1. Click "Create Bucket"
1. In "Bucket Name", specify the name of the S3 bucket you'd like to use.  Click "Next"
1. Under "Configure Options", select "Keep all versions of an object in the same bucket." and "Automatically encrypt objects when they are stored in S3." (and any other options you would like to use).  Click "Next"
1. Under permissions, select options to keep the bucket as restricted as possible.  Click "Next"
1. Click "Create Bucket"

#### Create an IAM policy for the bucket

1. In your AWS account, navigate to IAM
1. Click "Policies" (on the left nav bar)
1. Click "Create policy"
1. Click "JSON".  
1. In the JSON field editor, add this (replace `BUCKET_NAME` with your bucket name):

  ```json
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Effect": "Allow",
              "Action": "s3:ListAllMyBuckets",
              "Resource": "arn:aws:s3:::*"
          },
          {
              "Effect": "Allow",
              "Action": "s3:*",
              "Resource": [
                  "arn:aws:s3:::BUCKET_NAME",
                  "arn:aws:s3:::BUCKET_NAME/*"
              ]
          }
      ]
  }
  ```

1. Click "Review Policy"
1. Specify a name for your policy (for example, `justinrlee-spinnaker-temp-s3`)

#### Create an IAM account

1. In your AWS account, navigate to IAM
1. Click on "Users", and "Add user"
1. Specify a unique, relevant user name (for example, `justinrlee-spinnaker-temp-s3`).
1. Select "programmatic access"
1. Click "Next"
1. Select the option "Attach existing policies directly"
1. Search for and select the policy that you created.
1. Click "Next: Tags"
1. Click "Next: Review"
1. Click "Create user"
1. Make sure to save the access key ID and Secret access key

### Configure Halyard to use the bucket

You must add the access key id and secret access key to the Halyard configuration, and configure Halyard to configure Spinnaker to use the bucket.

*Do this in the Halyard container*

```bash
# Replace with key id
export ACCESS_KEY_ID=XXX
export BUCKET_NAME=justinrlee-spinnaker-temp
export REGION=us-east-1

# Run this; it will prompt for the secret key
hal config storage s3 edit \
    --bucket ${BUCKET_NAME} \
    --access-key-id ${ACCESS_KEY_ID} \
    --secret-access-key \
    --region ${REGION}
```

You'll be prompted for the secret key, and the access will be validated.  Then, do this:

```bash
hal config storage edit --type s3
```

## Select the version of Spinnaker to install

Before Halyard will install Spinnaker, you should specify the version of Spinnaker you want to use.

You can get a list of available versions of spinnaker with this command:

```bash
hal version list
```

And then you can select the version with this:

```bash
# Replace with version of choice:
export VERSION=1.10.6
hal config version edit --version $VERSION
```

## Install Spinnaker

Now that your halconfig is completely configured for the initial Spinnaker, you can tell Halyard to actually install Spinnaker:

```bash
hal deploy apply
```

## Connect

It may take a little bit of time for your Spinnaker installation to complete.  You can monitor by running:

```bash
kubectl get pods -n ${NAMESPACE} --watch
```

In order to connect to your Spinnaker cluster, you can port forward directly from your workstation to your spinnaker cluster.  You must forward port 8084 to spin-gate (Spinnaker's API) and port 9000 to spin-deck (Spinnaker's UI)
```bash
# Replace with the namespace where Spinnaker was installed
export NAMESPACE="spinnaker-installation"
kubectl -n=${NAMESPACE} port-forward \
    $(kubectl -n=${NAMESPACE} get po -l=cluster=spin-deck -o=jsonpath='{.items[0].metadata.name}') 9000 &
kubectl -n=${NAMESPACE} port-forward \
    $(kubectl -n=${NAMESPACE} get po -l=cluster=spin-gate -o=jsonpath='{.items[0].metadata.name}') 8084 &
```