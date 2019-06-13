---
date: 2019-06-13
title: Run a generic shell script (in Kubernetes) with Spinnaker
categories:
   - Kubernetes
description: How to run generic shell scripts in Spinnaker, using Kubernetes jobs.
type: Document
---

### Background

Spinnaker is a tool specialized in doing Deployments _natively_, this means without having to write shell scripts. That said, there are times when you need to run some custom logic related with your deployments. Spinnaker has a stage called `Script`, however this relies on a Jenkins job under hood and has been deprecated in favor of Jenkins stage.

If you want to run a script then Kubernetes jobs can be used to do that, but then you need to develop a docker container that has your script. A solution for this is to use a generic docker container and mount the script in a Kubernetes `ConfigMap`. Then you can define the script contents as part of the deployment yaml for the ConfigMap. 

### Basic approach

Stage: `Deploy (Manifest)`

```
apiVersion: v1
data:
  script.sh: |-
    echo "Hello world!"
    kubectl get pods
kind: ConfigMap
metadata:
  name: script-configmap
  namespace: german
---
apiVersion: batch/v1
kind: Job
metadata:
  labels:
    app: script-job
  name: script-job
  namespace: german
spec:
  backoffLimit: 2
  template:
    spec:
      containers:
        - command:
            - sh
            - /opt/script/script.sh
          image: 'bitnami/kubectl:1.12'
          name: script
          volumeMounts:
            - mountPath: /opt/script
              name: script-configmap
              readOnly: false
      restartPolicy: Never
      serviceAccountName: spinnaker-service-account
      volumes:
        - configMap:
            name: script-configmap
          name: script-configmap

```

Here we define a generic `Deploy (Manifest)` stage, supplying the above text contents. We define the ConfigMap as well as the Job. The script is part of the ConfigMap, and the Job runs a generic Docker container from docker hub that has the tool `kubectl`. We instruct the Job to execute a command, which happens to be the script which is mounted via the ConfigMap. In this case we have the possibility to edit the script without having to generate another Docker image. Another thing to consider is that jobs can be run only once in Kubernetes and are immutable, so we need to add a stage of `Delete (Manifest)` for deleting old jobs before our `Deploy (manifest)`:

![](/images/run-script-basic.png)

### Taking the shell script from version control

The basic approach is fine for small scripts, but writing something more complex inside a ConfigMap definition is no fun. Fortunately there's a trick in Spinnaker that allow us to read file contents from an artifact, which in turn can be a Github file, Bitbucket file, S3 file and any supported Spinnaker artifact that refers to text files.

We're going to use a `Webhook` stage for doing a request to Spinnaker's `clouddriver` service, which has already all the credentials needed for downloading artifacts ([you need to enable and supply those artifact credentials through halyard first](https://www.spinnaker.io/setup/artifacts/)). Clouddriver will in turn use the same logic it already uses in other parts of Spinnaker to download the file and return it to the `Webhook` stage. The output of this stage (and hence the file) is available in the pipeline execution context, so we can refer to it using [Pipeline Expressions](https://www.spinnaker.io/guides/user/pipeline/expressions/) in the ConfigMap definition:

![](/images/run-script-full-pipeline.png)

The configuration for Webhook stage is as follows:

```
Webhook URL:  http://spin-clouddriver:7002/artifacts/fetch
Method:       PUT
Payload:      {
                "artifactAccount": "<ACCOUNT NAME CONFIGURED IN HALYARD>",
                "reference": "https://api.github.com/repos/<ORG>/<REPO>/contents/<PATH TO FILE>",
                "type": "github/file",
                "version": "master"
              }
```

And the `Deploy (Manifest) - ConfigMap`:

```
apiVersion: v1
data:
  script.sh: '${#stage("Webhook")["context"]["webhook"]["body"]}'
kind: ConfigMap
metadata:
  name: script-configmap
  namespace: german
```

Then the `Run job` stage can just have the yaml description of the kubernetes job.

