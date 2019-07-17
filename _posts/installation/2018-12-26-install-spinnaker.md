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

This document will guide you through a manual installation of Spinnaker (either Open Source or Armory), in your Kubernetes cluster.

### Installation Architecture

Here are some things to keep in mind when installing Spinnaker:

* Spinnaker is composed of set of microservices (mostly Java-based)
* With this installation method, we will choose a namespace within your existing Kubernetes cluster and run the Spinnaker components as Kubernetes objects within your Kubernetes cluster.  This actually consists of the following:
  * Kubernetes *secrets*, which have Spinnaker configurations and credentials
  * Kubernetes *deployments*, which create *replicasets* which create the *pods* that actually comprise the Spinnaker components
  * Kubernetes *services*, which set up ClusterIP load balancers for the Spinnaker microservices to talk with each other.
* Think of the Kubernetes cluster where Spinnaker is installed as the **Installation Target**
* Spinnaker is used to deploy to multiple cloud environments, including Kubernetes.  Think of these as **Depoyment Targets**.  
* The cluster where Spinnaker is installed (the **Installation Target**) will also be one of your **Deployment Targets**.
* Spinnaker will need to be able to access the API for each of your Deployment Targets, and will need credentials (either provided directly or via inherited instance permissions) for each of your Deployment Targets (including the Installation Target)
* Spinnaker is configured and managed with a tool called Halyard, which consists of the following:
  * A Halyard daemon, which listens for API calls from the Halyard CLI and actually creates the Kubernetes objects that make up Spinnaker.
  * A Halyard CLI tool (`hal`), which is used to interact with the Halyard daemon
    * *You use commands such as `hal config provider kubernetes enable` to make changes to the Halyard configuration and `hal deploy apply` to apply and deploy your changes to the KUbernetes cluster.*
  * A yaml-based Halyard configuration file (`~/.hal/config`, also known as the **halconfig**), which has the configuration for your cluster
  * A set of additional yaml files within the `~/.hal` directory and subdirectories, which can be used to customize your Spinnaker installation
* In addition to the Kubernetes cluster where Spinnaker is installed, Spinnaker needs the following persistent state stores:
  * An S3 or S3-compatible endpoint (S3, GCS, AZS, Minio, etc.) in which to store persistent configuration.
    * This can be internal or external to the Kubernetes cluster; in this document, we will use an S3 bucket.
  * A Redis or Redis-compatible endpoint (Elasticache, Memorystore, etc.) both to use for execution history and to use as a cache.
    * This can be internal or external to the Kubernetes cluster; in this document, Halyard will default to creating a `redis` container alongside the Spinnaker microservices, and Spinnaker will use that.
* Halyard can be run in multiple places; it only needs the following:
  * API access to your Kubernetes cluster
  * Credentials for your various cloud environments.
* In this document, we will run Halyard in a Docker container from outside the cluster, and it will interact with your pre-existing Kubernetes cluster.
  * Halyard can be run locally on your workstation, or on some persistent virtual machine such as an EC2 instance that has access to your Kubernetes cluster.
  * You should both preserve and keep secret the full contents of everything in your `.hal` directory and all other directories mounted into your Halyard container.

This document will guide you through the initial installation of Open Source or Armory Spinnaker, in the following environment:

* Spinnaker installed in Kubernetes
* Using an S3 bucket as the configuration store
* Installed via Halyard, run as a Docker container
  * This `halyard` container will have multiple volume mounts added to it, discussed below.

We'll do this by starting a Halyard docker container (with a container name of `oss-halyard`), and then interacting with the Halyard daemon in that container from another shell session.

Later articles may cover the following:

* Configuring Spinnaker to use an external Redis (rather than the default redis Kubernetes container)
* Configuring high availability
* Adding a Kubernetes account ([document here](2018-12-26-spinnaker-add-kubernetes.md))
* Adding an AWS account

## Pre-requisites

You must have the following items available prior to installing Spinnaker:
* A place to run Halyard.  This can be any place a Docker container can run.  Here are some options:
  * On a Linux or OSX workstation (Windows should work, but permissions for volumes can be complicated)
  * On an EC2 instance with Docker installed.
* An S3 bucket, and IAM permissions to list the bucket and read/write to the S3 bucket, via one of the following:
  * IAM credentials in the form of an AWS Access Key and Secret Accesss Key
  * IAM instance role attached to the Kubernetes nodes where Spinnaker will be running
* Credentials to your Kubernetes cluster (a *kubeconfig*) that can be used from within a Docker container.  If you're using IAM credentials for an EKS cluster, we will copy these IAM credentials into the Docker container (in `~/.aws`), so EKS credentials *should* work.

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

## Installation
The next several sections will walk through the installation of Spinnaker on your Kubernetes cluster.

## Start the Halyard container

Halyard can be run directly on a Linux machine or on OSX, but is somewhat more portable in a Docker container.  In order to to this, you should directly mount three directories into your docker container. It's useful to have these directories accessible from a shared root.  We also recommend adding an additional directory to store sensitive information passed to Spinnaker (Spinnaker secret management forthcoming):

* `.hal` (where all Halyard-specific configurations are stored)
* `.aws` (where AWS credentials, in the event that you're accessing Kubernetes via EKS IAM credentials, are stored)
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
export ARMORY_HALYARD_IMAGE=armory/halyard-armory:1.3.1
export OSS_HALYARD_IMAGE=gcr.io/spinnaker-marketplace/halyard:stable

# If you want to install OSS halyard, switch these out
export HALYARD_IMAGE=${ARMORY_HALYARD_IMAGE}
# export HALYARD_IMAGE=${OSS_HALYARD_IMAGE}

docker run --name halyard -it --rm \
  -v ${PWD}/.hal:/home/spinnaker/.hal \
  -v ${PWD}/.kube:/home/spinnaker/.kube \
  -v ${PWD}/.aws:/home/spinnaker/.aws \
  -v ${PWD}/.secret:/home/spinnaker/.secret \
  ${HALYARD_IMAGE}
```

Then, in a separate terminal/shell session, enter the docker container with this:

```bash
docker exec -it halyard bash

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

# Replace with the bucket you are using
export BUCKET_NAME=justinrlee-spinnaker-temp
# Replace with the AWS region where your bucket is
export REGION=us-east-1
# This is the default directory within the bucket that will be used by Spinnaker; feel free to change to a different directory.
export ROOT_FOLDER=front50
```

### Create the namespace to install Spinnaker in

If the namespace does not exist, you can create it.

*Do this in the Halyard container*

```bash
kubectl --context ${CONTEXT} create ns ${NAMESPACE}
```

### Create the service account and credentials

Create a Kubernetes manifest that contains the service account, role, and rolebinding.
*This will create two **roles** within the specified namespace, one with full access to the namespace and one with limited access to the namespace.  It will also bind both roles to the service account.  In the next step, we show how to remove the admin role binding, if you want fewer permissions*

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
- apiGroups: [""]
  resources: ["replicationcontrollers/scale"]
  verbs: ["get", "update"]
- apiGroups: ["extensions"]
  resources: ["deployments/scale", "replicasets/scale"]
  verbs: ["get", "update"]
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
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: admin-ROLE_NAME
  namespace: NAMESPACE
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: admin-ROLE_NAME-binding
  namespace: NAMESPACE
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: admin-ROLE_NAME
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

### Optionally, remove the admin role binding
The above set of roles and bindings will grant your service account full access to all items with the Spinnaker namespace.  If you don't want this set of permissions, you can remove the admin role binding:

```bash
kubectl --context ${CONTEXT} -n ${NAMESPACE} \
  delete rolebinding admin-${ROLE_NAME}-binding
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

### Configure Halyard to use the bucket

If you are using an access key id and secret access key for the Halyard configuration, configure Halyard to configure Spinnaker to use the bucket using the key and secret key

*Do this in the Halyard container*

```bash
export ACCESS_KEY_ID=XXX
# Run this; it will prompt for the secret key
hal config storage s3 edit \
    --bucket ${BUCKET_NAME} \
    --access-key-id ${ACCESS_KEY_ID} \
    --secret-access-key \
    --root-folder ${ROOT_FOLDER} \
    --region ${REGION}
```

You'll be prompted for the secret key, and the access will be validated.

Alternately, if you're using IAM instance roles/profiles directly attached to the nodes where Kubernetes is running, you can use this:
```bash
hal config storage s3 edit \
    --bucket ${BUCKET_NAME} \
    --root-folder ${ROOT_FOLDER} \
    --no-validate \
    --region ${REGION}
```

Either way, run this to configure Spinnaker to know to use S3:

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
# Replace with desired version from output above:
# Armory versions will start with 2.x.x
# OSS will start with 1.x.x
export VERSION=2.1.3
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

In order to connect to your Spinnaker cluster, you can port forward directly from your workstation to your spinnaker cluster.  **This must be run on your local workstation.** You must forward port 8084 to spin-gate (Spinnaker's API) and port 9000 to spin-deck (Spinnaker's UI)
```bash
# Replace with the namespace where Spinnaker was installed
export NAMESPACE="spinnaker-installation"
kubectl -n=${NAMESPACE} port-forward \
    $(kubectl -n=${NAMESPACE} get po -l=cluster=spin-deck -o=jsonpath='{.items[0].metadata.name}') 9000 &
kubectl -n=${NAMESPACE} port-forward \
    $(kubectl -n=${NAMESPACE} get po -l=cluster=spin-gate -o=jsonpath='{.items[0].metadata.name}') 8084 &
```

## Install the NGINX ingress controller
In order to expose Spinnaker to end users, you have perform the following actions:
* Expose the spin-deck (UI) Kubernetes service on some URL endpoint
* Expose the spin-gate (API) Kubernetes service on some URL endpoint
* Update Spinnaker (via Halyard) to be aware of the new endpoints

**For the rest of this document, we are assuming you're on EKS - if you're in some other environment, adapt the following to your organization's desired Kubernetes ingress mechanism**

We're going to use the NGINX ingress controller on EKS:

From the `workstation machine` (where `kubectl` is installed):

Install the NGINX ingress controller components:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
```

Install the NGINX ingress controller EKS-specific service:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/aws/service-l4.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/aws/patch-configmap-l4.yaml
```

## Set up the Ingress for `spin-deck` and `spin-gate`

Identify the URLs you will use to expose Spinnaker's UI and API.

```bash
# Replace with actual values
SPIN_DECK_ENDPOINT=spinnaker.some-url.com
SPIN_GATE_ENDPOINT=api.some-url.com
NAMESPACE=spinnaker
```

Create a Kubernetes Ingress manifest to expose spin-deck and spin-gate (change your hosts and )
```bash
tee spin-ingress.yaml <<-'EOF'
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: spin-ingress
  namespace: NAMESPACE
  labels:
    app: spin
    cluster: spin-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: SPIN_DECK_ENDPOINT
    http:
      paths:
      - backend:
          serviceName: spin-deck
          servicePort: 9000
        path: /
  - host: SPIN_GATE_ENDPOINT
    http:
      paths:
      - backend:
          serviceName: spin-gate
          servicePort: 8084
        path: /
EOF

sed -i.bak \
  -e "s|NAMESPACE|${NAMESPACE}|g" \
  -e "s|SPIN_DECK_ENDPOINT|${SPIN_DECK_ENDPOINT}|g" \
  -e "s|SPIN_GATE_ENDPOINT|${SPIN_GATE_ENDPOINT}|g" \
  spin-ingress.yaml
```

Create the Ingress
```bash
kubectl apply -f spin-ingress.yaml
```

## Configure Spinnaker to be aware of its endpoints

Spinnaker must be aware of its endpoints to work properly.

This should be done from the halyard container:

```bash
SPIN_DECK_ENDPOINT=spinnaker.some-url.com
SPIN_GATE_ENDPOINT=api.some-url.com
SPIN_DECK_URL=http://${SPIN_DECK_ENDPOINT}
SPIN_GATE_URL=http://${SPIN_GATE_ENDPOINT}

hal config security ui edit --override-base-url ${SPIN_DECK_URL}
hal config security api edit --override-base-url ${SPIN_GATE_URL}

hal deploy apply
```

## Set up DNS

Once the ingress is up (this may take some time), you can get the IP address for the ingress:

```bash
$ kubectl describe -n spinnaker ingress spinnaker-nginx-ingress
Name:             spinnaker-nginx-ingress
Namespace:        spinnaker
Address:          35.233.216.189
Default backend:  default-http-backend:80 (10.36.2.7:8080)
Rules:
  Host                    Path  Backends
  ----                    ----  --------
  spinnaker.some-url.com
                          /   spin-deck:9000 (<none>)
  api.some-url.com
                          /   spin-gate:8084 (<none>)
Annotations:
  kubectl.kubernetes.io/last-applied-configuration:  {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{"kubernetes.io/ingress.class":"nginx"},"name":"spinnaker-nginx-ingress","namespace":"spinnaker"},"spec":{"rules":[{"host":"spinnaker.some-url.com","http":{"paths":[{"backend":{"serviceName":"spin-deck","servicePort":9000},"path":"/"}]}},{"host":"api.some-url.com","http":{"paths":[{"backend":{"serviceName":"spin-gate","servicePort":8084},"path":"/"}]}}]}}

  kubernetes.io/ingress.class:  nginx
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  CREATE  28s   nginx-ingress-controller  Ingress spinnaker/spinnaker-nginx-ingress
  Normal  UPDATE  20s   nginx-ingress-controller  Ingress spinnaker/spinnaker-nginx-ingress
```

Set up DNS so that your two URLs point to the IP address for the ingress (in the above, configure `spinnaker.some-url.com` and `api.some-url.com` to point to `35.233.216.189`).  This can be done via whatever your organization uses for DNS.

## Configuring TLS Certificates

Configuration of TLS certificates for ingresses is often very organization-specific.  In general, you would want to do the following:
* Add certificate(s) so that your ingress controller can use them
* Configure the ingress(es) so that NGINX (or your ingress) terminates TLS using the certificate(s)
* Update Spinnaker to be aware of the new TLS endpoints (note `https` instead of `http`)

  ```bash
  SPIN_DECK_ENDPOINT=spinnaker.some-url.com
  SPIN_GATE_ENDPOINT=api.some-url.com
  SPIN_DECK_URL=https://${SPIN_DECK_ENDPOINT}
  SPIN_GATE_URL=https://${SPIN_GATE_ENDPOINT}

  hal config security ui edit --override-base-url ${SPIN_DECK_URL}
  hal config security api edit --override-base-url ${SPIN_GATE_URL}

  hal deploy apply
  ```

## Next Steps
Now that you have Spinnaker up and running, here are some of the next things you may want to do:
* Configuration of certificates to secure your cluster (see [this section](#configuring-tls-certificates) for notes on this)
* Configuration of Authentication/Authorization (see the [Open Source Spinnaker documentation](https://www.spinnaker.io/setup/security/))
* Add Kubernetes accounts to deploy applications to (see [this KB article](https://kb.armory.io/installation/spinnaker-add-kubernetes/))
* Add GCP accounts to deploy applications to (see the [Open Source Spinnaker documentation](https://www.spinnaker.io/setup/install/providers/gce/))
* Add AWS accounts to deploy applications to (see the [Open Source Spinnaker documentation](https://www.spinnaker.io/setup/install/providers/aws/))