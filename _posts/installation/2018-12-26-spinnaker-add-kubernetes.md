---
date: 2018-12-26
title: Adding a Kubernetes (v2) Cloud Provider
categories:
   - Installation
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

**This whole document should be followed from within the halyard container or machine where halyard is running**

## Set up bash parameters

First, we'll set up bash environment variables that will be used by later commands

```bash
# Specify the name of the kubernetes context that has permissions in your target cluster and namespace
# to create the service account.  To get context names, you can run "kubectl config get-contexts".
export CONTEXT="aws-armory-dev"

# Enter the namespace where you want the Spinnaker service account to live
export NAMESPACE="spinnaker-system"

# Enter the namespace that you want to deploy to.  This can already exist, or can be created.
export TARGET_NAMESPACES=(namespace-1 namespace-2)

# Enter the name of the service account you want to create in the target namespace.
export SERVICE_ACCOUNT_NAME="spinnaker"

# Enter the name of the role you want to create in the target namespace.
export ROLE_NAME="spinnaker-role"

# Enter the account name you want Spinnaker to use to identify the deployment target.
export ACCOUNT_NAME="spinnaker-dev"
```

## Create namespace(s) (if they do not exist)

If the namespaces do not exist, you can create them.

```bash
# Create the Spinnaker service account namespace
kubectl --context ${CONTEXT} create ns ${NAMESPACE}

# Create the target namespaces
for TARGET_NS in ${TARGET_NAMESPACES[@]}; do
   kubectl --context ${CONTEXT} create ns ${TARGET_NS}
done
```

## Create the service account and credentials

Create a Kubernetes manifest that contains the service account, role, and rolebinding.
*Note that this may not have all necessary permissions, depending on what you're deploying - feel free to add additional permissions to this account as neccessary*

```bash
# Create the service account manifest
tee ${NAMESPACE}-service-account.yml <<-'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: SERVICE_ACCOUNT_NAME
  namespace: NAMESPACE
EOF

# Create template for roles/rolebinding manifests
tee service-account-template.yml <<-'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ROLE_NAME
  namespace: TARGET_NS
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
# These permissions are necessary for halyard to operate. We use this role also to deploy Spinnaker itself.
- apiGroups: [""]
  resources: ["services/proxy", "pods/portforward"]
  verbs: ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ROLE_NAME-binding
  namespace: TARGET_NS
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
  namespace: TARGET_NS
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: admin-ROLE_NAME-binding
  namespace: TARGET_NS
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: admin-ROLE_NAME
subjects:
- namespace: NAMESPACE
  kind: ServiceAccount
  name: SERVICE_ACCOUNT_NAME
EOF

# Update the service account file
sed -i.bak \
  -e "s/NAMESPACE/${NAMESPACE}/g" \
  -e "s/SERVICE_ACCOUNT_NAME/${SERVICE_ACCOUNT_NAME}/g" \
  ${NAMESPACE}-service-account.yml

# Create the service account
kubectl --context ${CONTEXT} apply -f ${NAMESPACE}-service-account.yml

# For each target namespace, stamp out a Kubernetes manifest with the role and rolebinding
for TARGET_NS in ${TARGET_NAMESPACES[@]}; do
  sed \
    -e "s/NAMESPACE/${NAMESPACE}/g" \
    -e "s/TARGET_NS/${TARGET_NS}/g" \
    -e "s/SERVICE_ACCOUNT_NAME/${SERVICE_ACCOUNT_NAME}/g" \
    -e "s/ROLE_NAME/${ROLE_NAME}/g" \
    service-account-template.yml > ${TARGET_NS}-service-account.yml
done

# For each target namespace, apply the manifest
for TARGET_NS in ${TARGET_NAMESPACES[@]}; do
  kubectl --context ${CONTEXT} apply -f ${TARGET_NS}-service-account.yml
done
```

Optionally, if you don't want the service accounts to have full access to the namespace(s), remove the admin role bindings:
```bash
for TARGET_NS in ${TARGET_NAMESPACES[@]}; do
  kubectl --context ${CONTEXT} -n ${TARGET_NS} \
    delete rolebinding admin-${ROLE_NAME}-binding
done
```

## Create a minified kubeconfig

In order for Spinnaker to talk to a Kubernetes cluster, it must be provided a kubeconfig.  We're going to create a trimmed-down kubeconfig that only has the service account and token.

```bash

#################### Create minified kubeconfig
NEW_CONTEXT=${NAMESPACE}-sa
KUBECONFIG_FILE="kubeconfig-${NAMESPACE}-${CONTEXT}-sa"

SECRET_NAME=$(kubectl get serviceaccount ${SERVICE_ACCOUNT_NAME} \
  --context ${CONTEXT} \
  --namespace ${NAMESPACE} \
  -o jsonpath='{.secrets[0].name}')
TOKEN_DATA=$(kubectl get secret ${SECRET_NAME} \
  --context ${CONTEXT} \
  --namespace ${NAMESPACE} \
  -o jsonpath='{.data.token}')

# This is necessary to handle both OSX and bash base64, which have different flags
# Any errors on the first command can be ignored
TOKEN=$(echo ${TOKEN_DATA} | base64 -d)
if [[ ! $? -eq 0 ]]; then TOKEN=$(echo ${TOKEN_DATA} | base64 -D); fi

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

## Add the kubeconfig and cloud provider to Spinnaker
You should copy the kubeconfig to a place accessible to halyard; this choice is left to the reader, but one option is `~/.secrets/`, which should be mounted into your halyard container

```bash
# Feel free to reference a different location
KUBECONFIG_DIRECTORY=~/.secret/
cp ${KUBECONFIG_FILE} ${KUBECONFIG_DIRECTORY}
export KUBECONFIG_FULL=$(realpath ${KUBECONFIG_DIRECTORY}${KUBECONFIG_FILE})

# Enable the kubernetes provider - this is probably already be enabled, if Spinnaker is installed in Kubernetes
hal config provider kubernetes enable
# Enable artifacts; not strictly neccessary for Kubernetes but will be useful in general
hal config features edit --artifacts true

# Add account
hal config provider kubernetes account add ${ACCOUNT_NAME} \
  --provider-version v2 \
  --kubeconfig-file ${KUBECONFIG_FULL} \
  --namespaces ${NAMESPACE}

# Apply changes
hal deploy apply
```

## Done!
After your changes get applied, you should be able to see the new Kubernetes account in your Spinnaker UI and be able to deploy to it.

**Don't forget to clear your browser cache / hard refresh your browser (`cmd-shift-r` or `control-shift-r`)**