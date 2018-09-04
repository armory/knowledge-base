---
date: 2018-09-04
title: Exposing Spinnaker
categories:
   - Admin
description: Exposing Spinnaker
type: Document
---

_Note: This guide assumes you're deploying Spinnaker on Kubernetes using the [Distributed](https://www.spinnaker.io/setup/install/environment/#distributed-installation) deployment type with Halyard._

## How To
When deploying Spinnaker, one of the first things you'll want to do is expose it to your users. When using Halyard, you can eaily connect to your instance locally with `hal deploy connect`, however, this isn't scalable once you start rolling Spinnaker out. While there are many way to expose Spinnaker, we find the method described in this post to be the easiest way to get started. If your organization has other requirements, this post may be helpful as you start working through the process.


First, we'll start by creating `LoadBalancer` Services which will expose the API (Gate) and the UI (Deck) via a Load Balancer in your cloud provider (on AWS this would be an ELB, for example). We'll do this by running the commands below and creating the `spin-gate-public` and `spin-deck-public` Services.


_`NAMESPACE` is the Kubernetes namespace where your Spinnaker install is located. Halyard defaults to `spinnaker` unless explicitly overridden._

```
$ export NAMESPACE={namespace}
$ kubectl expose service -n ${NAMESPACE} spin-gate --type LoadBalancer \
	--port 8084 \
	--targetPort 8084 \
	--name spin-gate-public
$ kubectl expose service -n ${NAMESPACE} spin-deck --type LoadBalancer \
	--port 9000 \
	--targetPort 9000 \
	--name spin-gate-public
```

Once these Services have been created, we'll need to update our Spinnaker deployment so that the UI understands where the API is located. To do this, we'll use Halyard to override the base URL for both the API and the UI and then redeploy Spinnaker.

```
$ export NAMESPACE={namespace}
$ export API_URL=$(kubectl get svc -n $NAMESPACE spin-gate-public -o jsonpath='{.status.loadBalancer.ingress[0].hostname)
$ export UI_URL=$(kubectl get svc -n $NAMESPACE spin-deck-public -o jsonpath='{.status.loadBalancer.ingress[0].hostname)
$ hal config security api edit --override-base-url http://${API_URL}:8084
$ hal config security ui edit --override-base-url http://${UI_URL}:9000
$ hal deploy apply
```

Now that we've successfully redeployed, we can now access our Spinnaker instance on the Load Balancer provisioned by Kubernetes. There are a couple of ways you could do this.

1. If you're on a Mac (OSX), you can run the following command to open a browser.
	
	```
	$ open http://${UI_URL}:9000
	```

2. If you're not on a Mac, use this command to get the url and then copy-and-paste it into your browser
   
   ```
   $ echo "http://${UI_URL}:9000"
   ```


## Cleanup

If you'd like to undo what we've done in the steps above you can simply delete the Kubernetes Services and reset the UI and API endpoints.

```
$ export NAMESPACE={namespace}
$ kubectl delete svc -n ${NAMESPACE} spin-gate-public spin-deck-public
$ hal config security api edit --override-base-url http://localhost:8084
$ hal config security ui edit --override-base-url http://localhost:9000
$ hal deploy apply
```

## Bonus

If you're running Kubernetes on AWS and wish to use Internal Load Balancers instead of public ELBs, you can add the following annotation the Kubernetes services.

```
service.beta.kubernetes.io/aws-load-balancer-internal: "true"
```


***

### More Resources: 
* [Generated Service YAML for this guide](https://gist.github.com/ethanfrogers/fbd4ec5af5748134583ccf745d9ba552)
