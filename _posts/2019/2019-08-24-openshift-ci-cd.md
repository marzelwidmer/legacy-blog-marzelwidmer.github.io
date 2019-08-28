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
oc new-project development --display-name="Development Environment"
oc new-project testing --display-name="Testing Environment"    
oc new-project production --display-name="Production Environment"    
oc new-project jenkins --display-name="Jenkins CI/CD"  
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
    - type: ConfigChange" | oc create -f - -n jenkins
```

## Create new Jenkins App with Persistent
Creating an Application From Source Code. `jenkins` is already available in the templates of `OKD`

```bash
oc new-app jenkins-persistent --name jenkins --param ENABLE_OAUTH=true --param MEMORY_LIMIT=2Gi --param VOLUME_CAPACITY=4Gi -n jenkins
```

## Print out Route
With the following command we can get the URL for our Jenkins.
```bash 
oc get routes
```

```bash
echo apiVersion: v1
kind: BuildConfig
metadata:
  creationTimestamp: null
  labels:
    app: pipeline
    name: pipeline
  name: pipeline
spec:
  output: {}
  postCommit: {}
  resources: {}
  runPolicy: Serial
  source:
    type: None
  strategy:
    jenkinsPipelineStrategy:
      jenkinsfile: "node('maven') {\n  stage 'build & deploy in dev'\n  openshiftBuild(namespace:
        'development',\n  \t\t\t    buildConfig: 'myapp',\n\t\t\t    showBuildLogs:
        'true',\n\t\t\t    waitTime: '3000000')\n  \n  stage 'verify deploy in dev'\n
        \ openshiftVerifyDeployment(namespace: 'development',\n\t\t\t\t       depCfg:
        'myapp',\n\t\t\t\t       replicaCount:'1',\n\t\t\t\t       verifyReplicaCount:
        'true',\n\t\t\t\t       waitTime: '300000')\n  \n  stage 'deploy in test'\n
        \ openshiftTag(namespace: 'development',\n  \t\t\t  sourceStream: 'myapp',\n\t\t\t
        \ sourceTag: 'latest',\n\t\t\t  destinationStream: 'myapp',\n\t\t\t  destinationTag:
        'promoteQA')\n  \n  openshiftDeploy(namespace: 'testing',\n  \t\t\t     deploymentConfig:
        'myapp',\n\t\t\t     waitTime: '300000')\n\n  openshiftScale(namespace: 'testing',\n
        \ \t\t\t     deploymentConfig: 'myapp',\n\t\t\t     waitTime: '300000',\n\t\t\t
        \    replicaCount: '2')\n  \n  stage 'verify deploy in test'\n  openshiftVerifyDeployment(namespace:
        'testing',\n\t\t\t\t       depCfg: 'myapp',\n\t\t\t\t       replicaCount:'2',\n\t\t\t\t
        \      verifyReplicaCount: 'true',\n\t\t\t\t       waitTime: '300000')\n  \n
        \ stage 'Deploy to production'\n  timeout(time: 2, unit: 'DAYS') {\n      input
        message: 'Approve to production?'\n }\n\n  openshiftTag(namespace:
        'development',\n  \t\t\t  sourceStream: 'myapp',\n\t\t\t  sourceTag: 'promoteQA',\n\t\t\t
        \ destinationStream: 'myapp',\n\t\t\t  destinationTag: 'promotePRD')\n\n  \n
        \ openshiftDeploy(namespace: 'production',\n  \t\t\t     deploymentConfig:
        'myapp',\n\t\t\t     waitTime: '300000')\n  \n  openshiftScale(namespace:
        'production',\n  \t\t\t     deploymentConfig: 'myapp',\n\t\t\t     waitTime:
        '300000',\n\t\t\t     replicaCount: '2')\n  \n  stage 'verify deploy in production'\n
        \ openshiftVerifyDeployment(namespace: 'production',\n\t\t\t\t       depCfg:
        'myapp',\n\t\t\t\t       replicaCount:'2',\n\t\t\t\t       verifyReplicaCount:
        'true',\n\t\t\t\t       waitTime: '300000')\n}"
    type: JenkinsPipeline
  triggers:
  - github:
      secret: secret101
    type: GitHub
  - generic:
      secret: secret101
    type: Generic
status:
  lastVersion: 0 |  oc create -f - -n jenkins
```








## Clean Up Jenkins 
#### Search all with selector `--selector app=jenkins`
```
oc get all --selector app=jenkins
oc get all --selector app=jenkins-maven-slave
```
#### Delete jenkins with `--selector app=jenkins`
```
oc delete all --selector app=jenkins
```
#### Search jenkins ImageStream
``` 
oc get is -n jenkins
```
#### Delete jenkins ImageStream
``` 
oc delete is/jenkins -n jenkins
```


