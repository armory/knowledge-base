---
date: 2018-12-26
title: Adding a Kubernetes (v2) Cloud Provider
categories:
   - Install Guide
description: Adding a Kubernetes (v2) Cloud Provider
type: Document
---

# Creating and Adding a Kubernetes Spinnaker Account (Cloud Provider / Deployment Target)

Once you have (OSS or Armory) Spinnaker up and running, you'll want to start adding deployment targets.  *(This document assumes Spinnaker was installed with halyard, that you have access to your current halconfig (and a way to operate `hal` against it), and that you have a kubeconfig and `kubectl` with permissions to create the relevant Kubernetes entities (`service account`, `role`, and `rolebinding`)*

This document will guide you through the following:

* Creating the namespace to deploy to (if it does not yet exist)
* Creating a service account (and associated role and rolebinding) with permissions to deploy to the namespace
* Create a minified kubeconfig containing only the service account
* Adding the account to Spinnaker

## Set up bash parameters

First, we'll set up bash environment variables that will be used by later commands

```bash

# Specify name of the kubernetes context that has permissions to create the service account in your target cluster and namespace.  To get the list of contexts, you can run "kubectl config get-contexts"
export CONTEXT="aws-armory-dev"

# Enter the namespace that you want to deploy to.  This can already exist, or can be created.
export NAMESPACE="spinnaker-target-deleteafter-20181230"

# Enter the name of the service account you want to create.  This will be created in the target namespace
export SERVICE_ACCOUNT_NAME="spinnaker"

# Enter the name of the role you want to create.  This will be created in the target namespace
export ROLE_NAME="spinnaker-role"

# Enter the name you want Spinnaker to use to identify the deployment target
export ACCOUNT_NAME="dev-target"
```

## Create namespace (if it does not exist)

If the namespace does not exist, you can create it.

```bash
kubectl --context ${CONTEXT} create ns ${NAMESPACE}
```

## Create the service account and credentials

Create a Kubernetes manifest that contains the service account, role, and rolebinding.
*Note that this may not have all necessary permissions, depending on what you're deploying - feel free to add additional permissions to this account as neccessary*

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
# These permissions are necessary for halyard to operate. We use this role also to deploy Spinnaker itself.
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

## Create a minified kubeconfig

In order for Spinnaker to talk to a Kubernetes cluster, it must be provided a kubeconfig.  We're going to create a trimmed-down kubeconfig that only has the service account and token.

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

# This is necessary to handle both OSX and bash base64, which have different flags - any errors on the first command can be ignored
TOKEN=$(echo ${TOKEN_DATA} | base64 -d)
if [[ ! $? -eq 0 ]]; then TOKEN=$(echo ${TOKEN_DATA} | base64 -D); fi

# Create kubeconfig
kubectl config view --flatten --minify > ${KUBECONFIG_FILE}.tmp
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
rm ${KUBECONFIG_FILE}.tmp
```

## Add the kubeconfig and cloud provider to Spinnaker
You should copy the kubeconfig to a place accessible to halyard; this choice is left to the reader, but one option is `~/.hal/dependencies/`, which should travel with halyard

```bash
# Feel free to reference a different location
KUBECONFIG_DIRECTORY="~/.hal/dependencies/"
cp ${KUBECONFIG_FILE} ${KUBECONFIG_DIRECTORY}

# Enable the kubernetes provider - is probably already be enabled, if Spinnaker is installed in Kubernetes
hal config provider kubernetes enable
# Enable artifacts; not strictly neccessary for Kubernetes but will be useful in general
hal config features edit --artifacts true

# Add account
hal config provider kubernetes account add ${ACCOUNT_NAME} \
  --provider-version v2 \
  --kubeconfig-file ${KUBECONFIG_FILE} \
  --namespaces ${NAMESPACE}

# Apply changes
hal deploy apply
```
