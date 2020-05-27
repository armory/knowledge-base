---
date: 2020-05-27
title: How to fix TLS error "Reason: extension (5) should not be presented in certificate_request"
categories:
   - troubleshooting
   - tls
description: How to fix TLS error "Reason: extension (5) should not be presented in certificate_request"
type: Document
---


# Mounting Custom CA to Operator Spinnaker & Halyard in Operator

Keyword: “x509: certificate signed by unknown authority”

## Background: 

This situation occurred with a custom cert on the vault server for secrets. 

```
    error fetching vault token - error logging into vault using kubernetes auth: Put https://spinnaker-vault.vault.svc.cluster.local:8200/v1/auth/kubernetes/login: x509: certificate signed by unknown authority
```

In order to do so - we first need to copy the original cert.pem file from /etc/ssl/cert.pem and then append the custom ca for vault, and then mount the cert.pem similar to the /etc/ssl/certs/java/cacerts

## Instructions:
after spinnaker-operator launches

1. cp cert.pem from the operator to local directory
2. `kubectl cp -n spinnaker-operator -c spinnaker-operator spinnaker-operator-xxxxx:etc/ssl/cert.pem .`
3. append cert to cert.pem
4. `cat myCA.crt >> cert.pem`
5. `cat myVault.crt >> cert.pem` ... and any other additional crt
6. create a secret for cert.pem to be mounted into spinnaker operator
7. `kubectl create secret generic spinop-certs -n spinnaker-operator --from-file cert.pem`
8. add volume and volume mounts to Spinnaker Operator deployment

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