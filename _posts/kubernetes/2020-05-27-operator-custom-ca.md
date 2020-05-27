---
date: 2020-05-27
title: How to add a custom CA for Operator and Vault
categories:
   - kubernetes
   - tls
description: Enable the Operator and Vault to use a custom CA.
type: Document
---

## Problem

When Spinnaker attempts to fetch a Vault token, you encounter the following error:

```
    error fetching vault token - error logging into vault using kubernetes auth: Put https://spinnaker-vault.vault.svc.cluster.local:8200/v1/auth/kubernetes/login: x509: certificate signed by unknown authority
```

This happens because you are using a custom CA and the Operator.

## Solution

Resolving this issue involves the following steps: 
* Copy the original `cert.pem` file from `/etc/ssl/cert.pem` a
* Append the custom ca for Vault
* Mount the `cert.pem` similar to the `/etc/ssl/certs/java/cacerts`

While the Operator is running, perform the following steps:

1. Copy `cert.pem` from the Operator to a local directory:
   
   ```
   kubectl cp -n spinnaker-operator -c spinnaker-operator spinnaker-operator-xxxxx:etc/ssl/cert.pem .
   ```

2. Append your custom cert to `cert.pem`:
   ```
   cat myCA.crt >> cert.pem
   ```

3. Append any other additional certs:
   ```
   cat myVault.crt >> cert.pem
   ```
4. Create a Kubernetes secret for `cert.pem`:
   ```
   kubectl create secret generic spinop-certs -n spinnaker-operator --from-file cert.pem
   ```
   This secret gets mounted into Spinnaker Operator.
5. Add volume and volume mounts to Spinnaker Operator:

   ```
   spec:
     serviceAccountName: spinnaker-operator
     containers:
       - name: spinnaker-operator
         image: armory/armory-operator:0.4.0
         ...
           - mountPath: /etc/ssl
             name: internal-cert-pem
       - name: halyard
         image: armory/halyard-armory:operator-0.4.x
         imagePullPolicy: IfNotPresent
         ...
         volumeMounts:
           - mountPath: /etc/ssl/certs/java
             name: internal-trust-store
     volumes:
     - name: internal-trust-store
       secret:
         defaultMode: 420
         secretName: internal-trust-store
     - name: internal-cert-pem
       secret:
         defaultMode: 420
         secretName: internal-cert-pem
   ```