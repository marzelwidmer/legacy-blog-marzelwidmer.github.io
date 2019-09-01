---
layout: post
title: Openshift - Distributed Tracing System
date: 2019-09-01
description: Distributed Tracing System # Add post description (optional)
img: 2019/openshift-distributed-tracing-system/101010.jpg  # Add image post (optional)
tags: [Blog, Jaeger, Openshift, OKD, Monitoring, Distributed Tracing]
author: # Add name author (optional)
---
As on-the-ground microservice practitioners are quickly realizing, the majority of operational problems that arise when moving to a
distributed architecture are ultimately grounded in two areas: networking and observability. 
It is simply an orders of magnitude larger problem to network and debug a set of intertwined distributed services versus a single monolithic application.
![Distributed Tracing In A Nutshell](/assets/img/2019/openshift-distributed-tracing-system/Distributed-Tracing-In-A-Nutshell.png)

## Jaeger 
Jaeger is a open source, end-to-end distributed tracing Monitor and troubleshoot transactions in complex distributed systems.
[CNCF Webinar Intro Jaeger](https://www.cncf.io/wp-content/uploads/2018/01/CNCF_Webinar_Intro_Jaeger_v1.0_-_2018-01-16.pdf){:target="_blank"}

Based on : [jaegertracing.io](https://www.jaegertracing.io/docs/1.13/operator/#installing-the-operator-on-okd-openshift){:target="_blank"}

## Installing the Operator on OKD/OpenShift
Login in with privileged user `oc login -u <privileged user>` and create a `monitoring` project to install the operator.
``` 
oc new-project monitoring
```

Deploy [CustomResourceDefinition](/assets/img/2019/openshift-distributed-tracing-system/deploy/jaegertracing_v1_jaeger_crd.yaml){:target="_blank"}
[ServiceAccount](/assets/img/2019/openshift-distributed-tracing-system/deploy/service_account.yaml){:target="_blank"} [ClusterRole](/assets/img/2019/openshift-distributed-tracing-system/deploy/role.yaml){:target="_blank"} 
[ClusterRoleBinding](/assets/img/2019/openshift-distributed-tracing-system/deploy/role_binding.yaml){:target="_blank"} [Operator](/assets/img/2019/openshift-distributed-tracing-system/deploy/operator.yaml){:target="_blank"}

``` 
oc create -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-distributed-tracing-system/deploy/jaegertracing_v1_jaeger_crd.yaml
oc create -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-distributed-tracing-system/deploy/service_account.yaml    
oc create -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-distributed-tracing-system/deploy/role.yaml    
oc create -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-distributed-tracing-system/deploy/role_binding.yaml    
oc create -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-distributed-tracing-system/deploy/operator.yaml    

```



https://www.jaegertracing.io/docs/1.13/

https://github.com/jaegertracing/jaeger-openshift 

https://github.com/jaegertracing/jaeger-operator

https://www.cncf.io/wp-content/uploads/2018/01/CNCF_Webinar_Intro_Jaeger_v1.0_-_2018-01-16.pdf





[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
