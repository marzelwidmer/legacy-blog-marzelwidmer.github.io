---
layout: post
title: Openshift - Jenkins Maven Fabric8 Delivery Pipeline
date: 2019-09-08
description: # Add post description (optional)
img: 2019/openshift-jenkins-maven-fabric8-delivery-pipeline/docker2.png  # Add image post (optional)
tags: [Blog, Jenkins, OpenShift, OKD, CI/CD, Docker, Images, Container, Release, Versioning, Tagging]
author: # Add name author (optional)
---


# Table of contents
* [Setup Deployment](#SetupDeployment)
* [Jenkins Pipeline](#JenkinsPipeline)
* [WebHooks](#WebHooks)


## Setup Deployment <a name="SetupDeployment"></a>
```bash
$ oc new-project development --display-name="Development Environment"
```

Deploy application with the `maven.fabric8.io` plugin in  `development` stage from local machine.
```bash
$ mvn fabric8:deploy -Dfabric8.namespace=development
```

Create `testing` project and setup the roles.
```bash
$ oc new-project testing --display-name="Testing Environment" 
$ oc policy add-role-to-user edit system:serviceaccount:jenkins:jenkins -n testing
$ oc policy add-role-to-group system:image-puller system:serviceaccounts:testing  \
          -n development
```

Create `DeploymentConfiguration` in `testing` stage.
```bash
$ oc create dc customer-service --image=docker-registry.default.svc:5000/development/customer-service:promoteQA -n testing
$ oc patch dc/customer-service  -p \
     '{"spec":{"template":{"spec":{"containers":[{"name":"default-container","imagePullPolicy":"Always"}]}}}}' -n testing
$ oc expose dc customer-service -n testing --port=8080 
$ oc expose svc/customer-service -n testing
```

Create `production` project and setup the roles.
```bash
$ oc new-project production --display-name="Production Environment" 
$ oc policy add-role-to-user edit system:serviceaccount:jenkins:jenkins -n production
$ oc policy add-role-to-group system:image-puller system:serviceaccounts:production  \
          -n development
```
Create `DeploymentConfiguration` in `production` stage.
```bash
$ oc create dc customer-service --image=docker-registry.default.svc:5000/development/customer-service:promotePRD -n production
$ oc patch dc/customer-service  -p \
     '{"spec":{"template":{"spec":{"containers":[{"name":"default-container","imagePullPolicy":"Always"}]}}}}' -n production
$ oc expose dc customer-service -n production --port=8080
$ oc expose svc/customer-service -n production
```


## Jenkins Pipeline  <a name="JenkinsPipeline"></a>
Let's creat a Jenkins Pipeline for the `customer-service` in the project `jenkins`.
```bash
$ oc create -n jenkins -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-jenkins-maven-fabric8-delivery-pipeline/customer-service-pipeline.yaml
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

## Private Repository access with secrets
```bash
$ oc create secret generic user-at-github \
      --from-literal=username=machineuser \
      --from-literal=password=<accesstoken> \
      --type=kubernetes.io/basic-auth \
      -n jenkins
$ oc secrets link builder user-at-github \
    -n jenkins
$ oc annotate secret/user-at-github \
      'build.openshift.io/source-secret-match-uri-1=https://github.com/marzelwidmer/*' \
    -n jenkins
$ oc label secret user-at-github credential.sync.jenkins.openshift.io=true \
    -n jenkins
$ oc set build-secret bc/customer-service-pipeline user-at-github --source
```
set GITHUB_TOKEN <accesstoken>
git remote add origin https://jenkins:$GITHUB_TOKEN@github.com/marzelwidmer/customer-service.git




![sync.jenkins](/assets/img/2019/openshift-jenkins-maven-fabric8-delivery-pipeline/sync.jenkins.openshift.io.png)
![secret-at-github](/assets/img/2019/openshift-jenkins-maven-fabric8-delivery-pipeline/secret-user-at-github.png)



> **_References:_**  
>   [Jenkins Client Plugin](https://github.com/openshift/jenkins-client-plugin)
>   [Best Practices for Managing Docker Versions](https://www.youtube.com/watch?v=MqsG9-HEcTw) 
>   [CI/CD - A/B - OpenShift - Jenkins](https://dzone.com/articles/continuous-delivery-with-openshift-and-jenkins-ab)
>https://cookbook.openshift.org/building-and-deploying-from-source/how-can-i-build-from-a-private-repository-on-gitlab.html
>https://blog.openshift.com/private-git-repositories-part-3-personal-access-tokens/
>
>https://jenkins.io/blog/2018/05/16/pipelines-with-git-tags/
>
>https://jgitver.github.io/#_goal
>https://github.com/jgitver/jgitver-maven-plugin
>https://codito.in/semantic-commits-for-git
>https://github.com/fteem/git-semantic-commits
>
>
> https://wiki.jenkins.io/display/JENKINS/semantic-versioning-plugin
>   https://wiki.jenkins.io/display/JENKINS/Git+Parameter+Plugin
>   https://gist.github.com/arehmandev/736daba40a3e1ef1fbe939c6674d7da8
>   https://wilsonmar.github.io/jenkins2-pipeline/
>   https://jenkins-jenkins.apps.c3smonkey.ch/job/jenkins/job/jenkins-customer-service-pipeline/pipeline-syntax/gdsl






