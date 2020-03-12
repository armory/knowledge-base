---
date: 2020-03-12
title: "Upgrade Kustomize version"
categories:
   - Kubernetes
description: How to update the Kustomize binary version mounting a volume.
type: Document
---

In order to change the kustomize binary from Rosco we can mount a volume and use an initContainer in order to download the desired version and replace the original with the new one.

Here's how to achieve this:

* Create the file `~/.hal/default/service-settings/rosco.yml` and set a volume; in this case we can use an _emptyDir_ type and in the **mounthPath** we should specify the directory where the kustomize binary exist and will be replaced. 

```yml
kubernetes:
  volumes:
  - id: shared-volume
    type: emptyDir
    mountPath: /packer/kustomize
```

* initContainer: modify `~/.hal/config` adding this *initContainer*, be sure to replace the _URL_ in the **args** attribute for the specific kustomize version that you want to use, and the volume's name should be the same as the voulme's id in the `rosco.yml`.

```yml

initContainers:
      rosco:
      - name: kustomize-binary
        image: tutum/curl:latest
        command: ["/bin/sh", "-c"]
        args: ["curl -s -L https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv3.5.4/kustomize_v3.5.4_linux_amd64.tar.gz | tar xvz -C /tmp"]
        volumeMounts:
        - name: shared-volume
          mountPath: /tmp
```

Apply the changes: `hal deploy apply` 

* Validation: Enter into the rosco pod `kubectl exec -it <rosco-pod> bash` then go to the kustomize directory `cd /packer/kustomize` and you must see the new binary then you can check the version

```bash
bash-5.0$ pwd
/packer/kustomize
bash-5.0$ ./kustomize version
{Version:kustomize/v3.5.4 GitCommit:3af514fa9f85430f0c1557c4a0291e62112ab026 BuildDate:2020-01-11T03:12:59Z GoOs:linux GoArch:amd64}
```