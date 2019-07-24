---
date: 2018-12-27
title: Reducing kubeconfig size
categories:
   - troubleshooting
description: Reducing kubeconfig size
type: Document
---

If you're seeing issues with halyard taking forever or complaining that your kubeconfig is too large, it may be worth reducing your kubeconfig or removing unnecessary certificate authorities from your kubeconfig (this has to do with the way kubeconfig yaml files are parsed).

You can flatten/minify a kubeconfig file with this:

```bash
kubeconfig config view --flatten --minify > NEW_KUBECONFIG_FILE
```

Alternately, if you have a full certificate authority bundle in your kubeconfig, your kubeconfig may be too large for halyard to process.  You can remove unnecessary certificate authorities with something like this, which will test all of the certificate authorities and remove all but the one(s) that actually work with your Kubernetes API.

*This only works on Linux, not OSX, due to differences in OSX and Linux awk/csplit/etc.*

#### Script to remove unnecessary certificate authorities

```bash
#!/bin/bash
KUBECONFIG=$1
TMPDIR=$(mktemp -d -p ${PWD})
echo "Using temp directory ${TMPDIR}"
SPLIT=$(grep -n "certificate-authority-data" ${KUBECONFIG} | cut -f1 -d:)

# If the CA is on a different line from certificate-authority-data, change this to +1
SPLIT=$(expr ${SPLIT} + 0)

echo "Splitting kubeconfig into three files, centered at line ${SPLIT}"
head -n $(expr ${SPLIT} - 1) ${KUBECONFIG} > ${TMPDIR}/${KUBECONFIG}_top
tail -n +$(expr ${SPLIT} + 1) ${KUBECONFIG} > ${TMPDIR}/${KUBECONFIG}_bottom
awk -v rownum="${SPLIT}" 'NR==rownum{print $NF}' ${KUBECONFIG} | base64 -d > ${TMPDIR}/${KUBECONFIG}_ca
csplit -sk -f ${TMPDIR}/cert- ${TMPDIR}/${KUBECONFIG}_ca '/END/1' '{*}' &> /dev/null
echo "Found $(ls ${TMPDIR}/cert-* | wc -l) CAs, testing all of them ..."

SERVER=$(awk '/server: /{print $2}' ${KUBECONFIG})
for cert in ${TMPDIR}/cert-*;
  do curl -s --cacert ${cert} ${SERVER} &>/dev/null;
  if [[ $? -eq 0 ]]; then
    ACTUAL=${cert}
    echo "Found valid CA:"
    cat ${ACTUAL}
  fi
done

# Adjust spacing based on indentation of yaml
echo "    certificate-authority-data: $(base64 -w0 ${ACTUAL})" > ${TMPDIR}/${KUBECONFIG}_middle

cat ${TMPDIR}/${KUBECONFIG}_top ${TMPDIR}/${KUBECONFIG}_middle ${TMPDIR}/${KUBECONFIG}_bottom > ${KUBECONFIG}.short
echo "Cleaning up and removing ${TMPDIR}"
if [[ -d ${TMPDIR} ]]; then rm -r ${TMPDIR}; fi
echo "New kubeconfig at ${KUBECONFIG}.short"
```

After running this, test your kubeconfig with `kubectl --kubeconfig <kubeconfig_file>.short get pods`.  Depending on the format of your starting kubeconfig,  you may need to tweak this script in one or more of the following ways:

* Add more or less spaces to the certificate-authority-data string
* If the `certificate-authority-data` contents is on the line following `certificate-authority-data` (vs. in the same line), change the SPLIT line to +1.

Sample usage:

```bash
b0bb885892b7:~/.secret/tmp spinnaker$ bash trim_kubeconfig.sh really-large-kubeconfig
Using temp directory /home/spinnaker/.secret/tmp/tmp.t52iSPT9Js
Splitting kubeconfig into three files, centered at line 4
Found 175 CAs, testing all of them ...
Found valid CA:
-----BEGIN CERTIFICATE-----
MIICyDCCAbCgAwIBAgIBADANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwprdWJl
cm5ldGVzMB4XDTE4MDgxNTIzMTU0MloXDTI4MDgxMjIzMTU0MlowFTETMBEGA1UE
AxMKa3ViZXJuZXRlczCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALgA
3fgJTXJ7LXCc8upBO1Z8cNl6uBOKMtmDVnjd/oOSAPsWxE7GbHsvATAsi4ratXA7
xhgCk/oY0U/r6yL1xKfkFLj8h8WdQbJG1xkxY/c+1+Azku8PuyVHYLe+s0Xgvo3G
ZctgBAEKRGTrj3yPU8aQr6JtIPzl2yZ+Xnbi7aR8WWBf8jEfGLlB44Od30WnfFCf
UebfjdH8e8wMSkthiWWv36gyDZlsgW0oIiyqWTRbhncCnbb/uqeZeBQH+sSe0VyH
4Fxbq18up2bte+pmrJet7aa6PCXcFrNn8GTei4WUYtrElVnVshUBKuXgTQMUJovX
91cwz8qmh/YHlTE+J3cCAwEAAaMjMCEwDgYDVR0PAQH/BAQDAgKkMA8GA1UdEwEB
/wQFMAMBAf8wDQYJKoZIhvcNAQELBQADggEBAAvcPI5mPLAa2+NOdV0Qhwk53s2u
zbTERdu6G85KCrInFvimrK2pcuX4tusXBNrKeiFf9YXlAeuIqiLrlXuNF7EUHRZW
Q5gAhYwmc87NCfYgKR2PbuBUvQUHFqZh1sBlwepIC5cdROxX4wDg0lFFjJ4tmzGi
NU5WSyDjhFJkpMLNSZXA/CjgZg1ZtvuNnox6O55VjAOKh+EcEku1wIdgWfIEBMWF
3gOLiUT1c6gHC96WQm0kWToR/E3BWO9+Rwvjs38+wa6Q00QJFxnjVq4Yy4T+Ib5A
+dTwrptRcyN9cSOKlCzKM1lmPX8qo3MpQ/QX1Op8Wa+uQi07Rof9WMVUzU4=
-----END CERTIFICATE-----

Cleaning up and removing /home/spinnaker/.secret/tmp/tmp.t52iSPT9Js
New kubeconfig at really-large-kubeconfig.short
b0bb885892b7:~/.secret/tmp spinnaker$
b0bb885892b7:~/.secret/tmp spinnaker$ ls -alh
total 380K
drwxr-xr-x  5 spinnaker spinnaker  160 Dec 27 14:44 .
drwxr-xr-x 21 spinnaker spinnaker  672 Dec 27 14:44 ..
-rw-r--r--  1 spinnaker spinnaker 372K Dec 27 13:06 really-large-kubeconfig
-rw-r--r--  1 spinnaker spinnaker 2.9K Dec 27 14:44 really-large-kubeconfig.short
-rw-r--r--  1 spinnaker spinnaker 1.4K Dec 27 14:44 trim_kubeconfig.sh
```
