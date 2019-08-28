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

## Install Jenkins 
You can create a Jenkins via Openshift Catalog in configure some more memory and volume capacity.

![Jenkins-From-Catalog-1](/images/posts/2019/openshift-ci-cd/Jenkins-from-catalog-1.png)
![Jenkins-From-Catalog-2](/images/posts/2019/openshift-ci-cd/Jenkins-from-catalog-2.png)
![Jenkins-From-Catalog-3](/images/posts/2019/openshift-ci-cd/Jenkins-from-catalog-3.png)

 
### Alternative Jenkins Installation 
You install the Jenkins with [ImageStream](https://docs.okd.io/latest/architecture/core_concepts/builds_and_image_streams.html)
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
``` 
oc new-app jenkins-persistent --name jenkins --param ENABLE_OAUTH=true --param MEMORY_LIMIT=2Gi --param VOLUME_CAPACITY=4Gi -n jenkins
```

## Get the Route (URL)
With the following command we can get the URL for our Jenkins.
```bash 
oc get routes
```

## Create Jenkins Pipeline
``` 
oc create -n jenkins -f \
    https://docs.marcelwidmer.org/images/posts/2019/openshift-ci-cd/pipeline.yaml
```

[jenkins-pipeline](/images/posts/2019/openshift-ci-cd/pipeline.yaml){:target="_blank"}

With the following command you can extract the BuildConfig `bc`   
``` 
oc get bc jenkins-pipeline -o yaml -n jenkins > jenkins-pipeline.yaml
```

Here you see how the Jenkins pipeline is configured.
```groovy
node('maven') {
  stage 'build & deploy in dev'
  openshiftBuild(namespace: 'development',
  			    buildConfig: 'myapp',
			    showBuildLogs: 'true',
			    waitTime: '3000000')
  
  stage 'verify deploy in dev'
  openshiftVerifyDeployment(namespace: 'development',
				       depCfg: 'myapp',
				       replicaCount:'1',
				       verifyReplicaCount: 'true',
				       waitTime: '300000')
  
  stage 'deploy in test'
  openshiftTag(namespace: 'development',
  			  sourceStream: 'myapp',
			  sourceTag: 'latest',
			  destinationStream: 'myapp',
			  destinationTag: 'promoteQA')
  
  openshiftDeploy(namespace: 'testing',
  			     deploymentConfig: 'myapp',
			     waitTime: '300000')

  openshiftScale(namespace: 'testing',
  			     deploymentConfig: 'myapp',
			     waitTime: '300000',
			     replicaCount: '2')
  
  stage 'verify deploy in test'
  openshiftVerifyDeployment(namespace: 'testing',
				       depCfg: 'myapp',
				       replicaCount:'2',
				       verifyReplicaCount: 'true',
				       waitTime: '300000')
  
  stage 'Deploy to production'
  timeout(time: 2, unit: 'DAYS') {
      input message: 'Approve to production?'
 }

  openshiftTag(namespace: 'development',
  			  sourceStream: 'myapp',
			  sourceTag: 'promoteQA',
			  destinationStream: 'myapp',
			  destinationTag: 'promotePRD')

  
  openshiftDeploy(namespace: 'production',
  			     deploymentConfig: 'myapp',
			     waitTime: '300000')
  
  openshiftScale(namespace: 'production',
  			     deploymentConfig: 'myapp',
			     waitTime: '300000',
			     replicaCount: '2')
  
  stage 'verify deploy in production'
  openshiftVerifyDeployment(namespace: 'production',
				       depCfg: 'myapp',
				       replicaCount:'2',
				       verifyReplicaCount: 'true',
				       waitTime: '300000')
}
```

## Add Role-Based Access Control
Let’s add in RBAC to our projects to allow the different service accounts to build, pro‐ mote, and tag images.
First we will allow the cicd project’s Jenkins service account edit access to all of our projects:

```
oc policy add-role-to-user edit system:serviceaccount:jenkins:jenkins  -n development
oc policy add-role-to-user edit system:serviceaccount:jenkins:jenkins  -n testing
oc policy add-role-to-user edit system:serviceaccount:jenkins:jenkins  -n production
oc policy add-role-to-group system:image-puller system:serviceaccounts:testing -n development
oc policy add-role-to-group system:image-puller system:serviceaccounts:production -n development
```


# Deploy
## Development
Let``s deploy a sample application in the development project. 
For this we change back to the development project with `oc project development`
```
oc project development
```

Creat a new app with `oc new-app`.

```
oc new-app --name=myapp \
  openshift/php:5.6~https://github.com/c3smonkey/cotd.git#master
```
Expose the application with the command `oc expose`
```
oc expose service myapp --name=myapp
```

Let``s see how the ImageStream look like with `oc get is -n development`
```
oc get is -n development

NAME    DOCKER REPO                                          TAGS     UPDATED
myapp   docker-registry.default.svc:5000/development/myapp   latest   3 minutes ago
```

## Testing
Change first to the testing project with `oc project testing`.
Then let``s create a deployment configuration with `oc create dc myapp --image=docker-registry.default.svc:5000/development/myapp:promoteQA`
The registry URL we get from the step before. on this deployment configuration we will add the promote preprod tag. This is configured in out Jenkins pipeline.
One change we need to make in testing is to update the imagePullPolicy for our container. 
By default, it is set to IfNotPresent, but we wish to always trigger a deployment when we tag a new image:
``` 
oc project testing
oc create dc myapp --image=docker-registry.default.svc:5000/development/myapp:promoteQA
oc patch dc/myapp -p \
    '{"spec":{"template":{"spec":{"containers":[{"name":"default-container","imagePullPolicy":"Always"}]}}}}'
```

Let``s also create our service and route while we’re at it (be sure to change the host‐name to suit your environment):
``` 
oc expose dc myapp --port=8080
oc expose service myapp --name=myapp
```


## Production
``` 
oc project production
oc create dc myapp --image=docker-registry.default.svc:5000/development/myapp:promotePRD
oc patch dc/myapp -p \
    '{"spec":{"template":{"spec":{"containers":[{"name":"default-container","imagePullPolicy":"Always"}]}}}}'
```


## Run Jenkins Pipeline
```
oc start-build jenkins-pipeline -n jenkins
```

![Pipeline-Prod-Deploy-Approval](/images/posts/2019/openshift-ci-cd/pipeline-prod-deploy-approval.png)
![Pipeline-Prod-Deploy](/images/posts/2019/openshift-ci-cd/pipeline-prod-deploy.png)


