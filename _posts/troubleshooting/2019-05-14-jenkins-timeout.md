---
date: 2019-05-14
title: Troubleshooting Jenkins timeout issues
categories:
   - troubleshooting
description: Troubleshooting Jenkins timeout issues
type: Document
---


If you're trying to hook up Jenkins with spinnaker and seeing timeout issues like this:
```
2019-05-14 19:31:28.366  WARN 1 --- [0.0-8088-exec-1] .m.m.a.ExceptionHandlerExceptionResolver : Resolved [com.netflix.hystrix.exception.HystrixRuntimeException: jenkins-build-qa-getJobs failed and no fallback available.]
2019-05-14 19:31:28.380  WARN 1 --- [0.0-8088-exec-3] .m.m.a.ExceptionHandlerExceptionResolver : Resolved [com.netflix.hystrix.exception.HystrixRuntimeException: jenkins-build-qa-getJobs failed and no fallback available.]
2019-05-14 19:31:29.842 ERROR 1 --- [RxIoScheduler-3] c.n.s.igor.jenkins.JenkinsBuildMonitor   : Failed to update monitor items for monitor=JenkinsBuildMonitor:partition=build-dev

com.netflix.hystrix.exception.HystrixRuntimeException: jenkins-build-dev-getProjects failed and no fallback available.
...
Caused by: retrofit.RetrofitError: connect timed out
    at retrofit.RetrofitError.networkError(RetrofitError.java:27) ~[retrofit-1.9.0.jar:na]
    at retrofit.RestAdapter$RestHandler.invokeRequest(RestAdapter.java:395) ~[retrofit-1.9.0.jar:na]
    at retrofit.RestAdapter$RestHandler.invoke(RestAdapter.java:240) ~[retrofit-1.9.0.jar:na]
    at com.sun.proxy.$Proxy93.getProjects(Unknown Source) ~[na:na]
    at sun.reflect.GeneratedMethodAccessor154.invoke(Unknown Source) ~[na:na]
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_201]
    at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_201]
    at com.netflix.spinnaker.kork.telemetry.InstrumentedProxy.invoke(InstrumentedProxy.java:87) ~[kork-telemetry-3.8.1-5814b41-stable145.jar:na]
    at com.sun.proxy.$Proxy93.getProjects(Unknown Source) ~[na:na]
    at com.netflix.spinnaker.igor.jenkins.client.JenkinsClient$getProjects.call(Unknown Source) ~[na:na]
    at com.netflix.spinnaker.igor.jenkins.service.JenkinsService$_getProjects_closure1.doCall(JenkinsService.groovy:78) ~[igor-web-1.1.1-63d06a5-stable160.jar:na]
...
Caused by: java.net.SocketTimeoutException: connect timed out
    at java.net.PlainSocketImpl.socketConnect(Native Method) ~[na:1.8.0_201]
    at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350) ~[na:1.8.0_201]
```

You as always, should check connectivity, exec into igor
`kubectl -n spinnaker exec -it <igor-pod-name> bash`
test for connectivity, for example testing for Armory Jenkins


```
bash-4.4$ nc -zv jenkins.armory.io 443
jenkins.armory.io (35.192.190.152:443) open
```

If the connectivity is not the issue, you can try to increase the timeout by modifying

profiles/igor-local.yml
```client:
    timeout: 60000

spinnaker:
  build:
    pollInterval: 300  # in seconds for all monitors
    pollingSafeguard:
       itemUpperThreshold: 4000
```

do a `hal deploy apply` and check if this solved your issue
