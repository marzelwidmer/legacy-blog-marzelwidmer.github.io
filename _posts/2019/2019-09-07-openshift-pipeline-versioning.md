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
Let's creat a Jenkins Pipeline for the `customer-service`.

```bash
$ oc create -n jenkins -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-pipeline-versioning/customer-service-pipeline.yaml
```


## WebHooks <a name="WebHooks"></a>
Here you can find how you creat a [WebHook](http://blog.marcelwidmer.org/openshift-delivey-pipeline/#WebHooks) to a public GitRepo. 



> **_References:_**  
>   [Best Practices for Managing Docker Versions](https://www.youtube.com/watch?v=MqsG9-HEcTw) 
>   [CI/CD - A/B - OpenShift - Jenkins](https://dzone.com/articles/continuous-delivery-with-openshift-and-jenkins-ab)


