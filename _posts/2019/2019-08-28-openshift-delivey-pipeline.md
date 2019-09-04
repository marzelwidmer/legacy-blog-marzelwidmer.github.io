---
layout: post
title: Openshift - Delivery Pipeline - Promoting Applications Across Environments
date: 2019-08-27 22:52:20
description: # Add post description (optional)
img: 2019/openshift-pipeline/delivey-pipeline.jpeg  # Add image post (optional)
tags: [Blog, Jenkins, Openshift, OKD, CI/CD]
author: # Add name author (optional)
---



![Pipeline-Multiproject-Example](/assets/img/2019/openshift-pipeline/promotion_pipeline.png)


## Create Project
We are going to use the CLI to create some projects. 
You can, of course, use the [console](https://console.c3smonkey.ch:8443/console/catalog){:target="_blank"} if you prefer. 
Let's create our projects first:

```
oc login  
oc new-project development \
    --display-name="Development Environment"
oc new-project testing \
    --display-name="Testing Environment"    
oc new-project production \
    --display-name="Production Environment"    
oc new-project jenkins \
    --display-name="Jenkins CI/CD"  
```

## Install Jenkins 
Create a Jenkins in thje `Jenkins CI/CD` project with some storage. Take the Jenkins form the catalog and set some more memory and volume capacity on it.
Everything else we let the default values. 

![Jenkins-From-Catalog-1](/assets/img/2019/openshift-pipeline/Jenkins-from-catalog-1.png)
![Jenkins-From-Catalog-2](/assets/img/2019/openshift-pipeline/Jenkins-from-catalog-2.png)
![Jenkins-From-Catalog-3](/assets/img/2019/openshift-pipeline/Jenkins-from-catalog-3.png)

[Jenkins BlueOcean](https://jenkins-jenkins.apps.c3smonkey.ch/blue/organizations/jenkins/)
 
## Create Jenkins Pipeline
``` 
oc create -n jenkins -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-pipeline/pipeline.yaml
```

[jenkins-pipeline](/assets/img/2019/openshift-pipeline/pipeline.yaml){:target="_blank"}

With the following command you can extract the BuildConfig `bc`   
``` 
oc get bc jenkins-pipeline -o yaml -n jenkins > jenkins-pipeline.yaml
```

Here you see how the Jenkins pipeline is configured.
There is also a other approach that you provide the [Jenkins Pipeline](#jenkins-pipeline-in-git) file in your Git-Repository.  
This I will cover later.

{% highlight groovy %}
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

{% endhighlight %}

## Add Role-Based Access Control
Let’s add in RBAC to our projects to allow the different service accounts to build, pro‐ mote, and tag images.
First we will allow the cicd project’s Jenkins service account edit access to all of our projects:

```
oc policy add-role-to-user edit system:serviceaccount:jenkins:jenkins \
      -n development
oc policy add-role-to-user edit system:serviceaccount:jenkins:jenkins \
    -n testing
oc policy add-role-to-user edit system:serviceaccount:jenkins:jenkins \
    -n production
```

This are needed for pulling the Images across the projects.
```
oc policy add-role-to-group system:image-puller system:serviceaccounts:testing \
    -n development
oc policy add-role-to-group system:image-puller system:serviceaccounts:production \
    -n development
```

# Deploy
## Development
Deploy a sample application in the development project. 
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

Take a look at the ImageStream with `oc get is -n development`
```
oc get is -n development

NAME    DOCKER REPO                                          TAGS     UPDATED
myapp   docker-registry.default.svc:5000/development/myapp   latest   3 minutes ago
```

## Testing
Change first to the testing project with `oc project testing`.
Then create a deployment configuration with `oc create dc myapp --image=docker-registry.default.svc:5000/development/myapp:promoteQA`
The registry URL we get from the step before. on this deployment configuration we will add the promote preprod tag. This is configured in out Jenkins pipeline.
One change we need to make in testing is to update the imagePullPolicy for our container. 
By default, it is set to IfNotPresent, but we wish to always trigger a deployment when we tag a new image:
``` 
oc project testing
oc create dc myapp --image=docker-registry.default.svc:5000/development/myapp:promoteQA
oc patch dc/myapp -p \
    '{"spec":{"template":{"spec":{"containers":[{"name":"default-container","imagePullPolicy":"Always"}]}}}}'
```

Create our service and route while we’re at it (be sure to change the host‐name to suit your environment):
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
oc expose service myapp --name=myapp
```


## Run Jenkins Pipeline
```
oc start-build jenkins-pipeline -n jenkins
```

![Pipeline-Prod-Deploy-Approval](/assets/img/2019/openshift-pipeline/pipeline-prod-deploy-approval.png)
![Pipeline-Prod-Deploy](/assets/img/2019/openshift-pipeline/pipeline-prod-deploy.png)


## Jenkins File <a name="jenkins-pipeline-in-git"></a>
Provide a Jenkins pipeline file in your Git repository. For this you can create in our root directory a file named `Jenkinsfile` something like
{% highlight groovy %}
pipeline {
  agent {
    node {
      label 'maven'
    }
  }
  stages {
    stage('Build') {
      steps {
        sh './mvnw clean install'
      }
    }
    stage('DeployDev') {
      steps {
        echo 'Deploy to Dev'
      }
    }
    stage('PromoteTest') {
      steps {
        echo 'Deploy to Test'
      }
    }
    stage('PromoteProd') {
      steps {
        echo 'Deploy to Prod'
      }
    }
  }
}
{% endhighlight %}

So now let's create a `BuildConfig` for our `catalaog-service` with the following [catalog-service-jenkins-pipeline](/assets/img/2019/openshift-pipeline/catalog-service-pipeline.yaml){:target="_blank"} configuration.

``` 
oc create -n jenkins -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-pipeline/catalog-service-pipeline.yaml
```




[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
