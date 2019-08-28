---
layout: post
title:	"Openshift CI/CD Pipeline"
date:	2019-08-24 12:00:00
categories:
    - blog
tags:
    - openshift
    - okd
    - jenkins
    - pipeline
    - ci/cd
    - hetzner
---


![Pipeline-Multiproject-Example](/images/posts/2019/openshift-ci-cd/Pipeline-Multiproject-Example.png)


## Create Project
We are going to use the CLI to create some projects. 
You can, of course, use the web-ui or your IDE if you prefer. 
Let’s create our projects first:

```
oc login  
oc new-project jenkins --display-name="Jenkins CI/CD"  
oc new-project development --display-name="Development Environment"
oc new-project testing --display-name="Testing Environment"    
oc new-project production --display-name="Production Environment"    
```

## Install ImageStream
Install Jenkins [ImageStream](https://docs.okd.io/latest/architecture/core_concepts/builds_and_image_streams.html)
with [skopeo](https://github.com/containers/skopeo) and Maven slave.

![ImageStreams](/images/posts/2019/openshift-ci-cd/ImageStreams.png)
```bash
echo "apiVersion: v1
kind: List
items:
- apiVersion: v1
  kind: ImageStream
  metadata:
    name: jenkins
    annotations:
      slave-label: maven-appdev
    labels:
      role: jenkins-slave
  spec:
    tags:
    - name: latest
- apiVersion: v1
  kind: BuildConfig
  metadata:
    annotations:
      description: Builds Jenkins slave image with maven
    labels:
      name: jenkins-maven-slave
      app: jenkins-slave
    name: jenkins-maven-slave
  spec:
    output:
      to:
        kind: ImageStreamTag
        name: jenkins-maven-slave:latest
    source:
      dockerfile: |
        FROM openshift/jenkins-slave-maven-centos7
        USER root
        RUN yum -y install skopeo apb && yum clean all
        USER 1001
    strategy:
      type: Docker
    triggers:
    - type: ConfigChange" | oc create -f -
```

## Create new Jenkins App with Persistent
Creating an Application From Source Code. `jenkins-persistent` is already available in the templates of `OKD`
```bash
oc new-app jenkins-persistent --param ENABLE_OAUTH=true --param MEMORY_LIMIT=2Gi --param VOLUME_CAPACITY=4Gi
```

## Print out Route
With the following command we can get the URL for our Jenkins.
```bash 
oc get routes
```
