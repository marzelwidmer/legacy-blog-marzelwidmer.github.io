---
layout: post
title: Openshift - Delivery Pipeline - Versioning - Tagging - Images
date: 2019-09-07
description: # Add post description (optional)
img: 2019/openshift-pipeline-versioning/docker.jpg  # Add image post (optional)
tags: [Blog, Jenkins, Openshift, OKD, CI/CD, Docker, Images, Container, Release, Versioning, Tagging]
author: # Add name author (optional)
---


# Table of contents
* [Jenkins Pipeline](#JenkinsPipeline)
* [WebHooks](#WebHooks)




## Jenkins Pipeline  <a name="JenkinsPipeline"></a>
Let's creat a Jenkins Pipeline for the `customer-service` in the project `jenkins`.
```bash
$ oc create -n jenkins -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-pipeline-versioning/customer-service-pipeline.yaml
```


## WebHooks <a name="WebHooks"></a>
How we can create a GitHub WebHook for a public Git repository take a look at the following post there we created already a  
[WebHook](http://blog.marcelwidmer.org/openshift-delivey-pipeline/#WebHooks) for the `catalog-service` but here some `oc` commands
for the `customer-service`.
```bash
$ oc set triggers bc/customer-service-pipeline --from-github  -n jenkins 
  buildconfig.build.openshift.io/customer-service-pipeline triggers updated
```
Grab the `Secret`.
```bash
$ oc get bc/customer-service-pipeline -n jenkins -o json | jq '.spec.triggers[].github.secret'
```
Grab `Webhook GitHub URL`. 
```bash
$ oc describe bc/customer-service-pipeline -n jenkins
```



> **_References:_**  
>   [Best Practices for Managing Docker Versions](https://www.youtube.com/watch?v=MqsG9-HEcTw) 
>   [CI/CD - A/B - OpenShift - Jenkins](https://dzone.com/articles/continuous-delivery-with-openshift-and-jenkins-ab)


