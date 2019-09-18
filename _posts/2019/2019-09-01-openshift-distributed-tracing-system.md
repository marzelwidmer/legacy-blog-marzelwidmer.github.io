---
layout: post
title: OpenShift - Distributed Tracing System
date: 2019-09-01
description: Distributed Tracing System # Add post description (optional)
img: 2019/openshift-distributed-tracing-system/101010.jpg  # Add image post (optional)
tags: [Blog, Jaeger, OpenShift, OKD, Monitoring, Distributed Tracing]
author: # Add name author (optional)
---
As on-the-ground microservice practitioners are quickly realizing, the majority of operational problems that arise when moving to a
distributed architecture are ultimately grounded in two areas: networking and observability. 
It is simply an orders of magnitude larger problem to network and debug a set of intertwined distributed services versus a single monolithic application.
![Distributed Tracing In A Nutshell](/assets/img/2019/openshift-distributed-tracing-system/Distributed-Tracing-In-A-Nutshell.png)

## Jaeger 
Jaeger is a open source, end-to-end distributed tracing Monitor and troubleshoot transactions in complex distributed systems.
[CNCF Webinar Intro Jaeger](https://www.cncf.io/wp-content/uploads/2018/01/CNCF_Webinar_Intro_Jaeger_v1.0_-_2018-01-16.pdf){:target="_blank"}

> **_NOTE:_**  Installing the operator on OKD Openshift [jaegertracing.io](https://www.jaegertracing.io/docs/1.13/operator/#installing-the-operator-on-okd-openshift){:target="_blank"}


## Installing the Operator on OKD/OpenShift
Login in with privileged user `oc login -u <privileged user>` and create a `monitoring` project to install the operator.
This creates the namespace used by default in the deployment files. If you want to install the Jaeger operator in a different namespace, 
you must edit the deployment files to change `monitoring` to the desired namespace value.
```bash
$ oc new-project monitoring
```

Deploy [CustomResourceDefinition](/assets/img/2019/openshift-distributed-tracing-system/deploy/jaegertracing_v1_jaeger_crd.yaml){:target="_blank"} for the `apiVersion: jaegertracing.io/v1`
[ServiceAccount](/assets/img/2019/openshift-distributed-tracing-system/deploy/service_account.yaml){:target="_blank"} [ClusterRole](/assets/img/2019/openshift-distributed-tracing-system/deploy/role.yaml){:target="_blank"} 
[ClusterRoleBinding](/assets/img/2019/openshift-distributed-tracing-system/deploy/role_binding.yaml){:target="_blank"} [Operator](/assets/img/2019/openshift-distributed-tracing-system/deploy/operator.yaml){:target="_blank"}

```bash
$ oc create -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-distributed-tracing-system/deploy/jaegertracing_v1_jaeger_crd.yaml
$ oc create -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-distributed-tracing-system/deploy/service_account.yaml    
$ oc create -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-distributed-tracing-system/deploy/role.yaml    
$ oc create -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-distributed-tracing-system/deploy/role_binding.yaml    
$ oc create -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-distributed-tracing-system/deploy/operator.yaml    

```

Grant the role `jaeger-operator` to users who should be able to install individual Jaeger instances. 
The following example creates a role binding allowing the user `developer` to create Jaeger instances:
```bash 
$ oc create \
    rolebinding developer-jaeger-operator \
    --role=jaeger-operator \
    --user=developer
```
After the role is granted, switch back to a non-privileged user.


# Quick Start - Deploying the AllInOne image
The simplest possible way to create a Jaeger instance is by creating a YAML file like the following example. 
This will install the default AllInOne strategy, which deploys the “all-in-one” image 
(agent, collector, query, ingestor, Jaeger UI) in a single pod, using in-memory storage by default.

> ⚠️ **Production installation**: For Production installation take a look at the official [production-strategy](https://www.jaegertracing.io/docs/1.13/operator/#production-strategy) documentation.


Login in with privileged user `oc login -u <privileged user>`
```bash 
$ echo "apiVersion: jaegertracing.io/v1                                                                                                                                     
  kind: Jaeger
  metadata:
    name: simplest" | oc create -f -
```
To get the pod name, query for the pods belonging to the `simplest` Jaeger instance:

```bash
$ oc get jaegers                                                                                                                                                            
NAME       AGE
simplest   10s
```

To get the routes:
```bash
$ oc get route

NAME       HOST/PORT                               PATH   SERVICES         PORT    TERMINATION   WILDCARD
simplest   simplest-monitoring.apps.c3smonkey.ch          simplest-query   <all>   reencrypt     None
```

Open the Browser and login with your Openshift credentials.

![Simples Jaeger Monitoring](/assets/img/2019/openshift-distributed-tracing-system/simplest-monitoring.png)



[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
