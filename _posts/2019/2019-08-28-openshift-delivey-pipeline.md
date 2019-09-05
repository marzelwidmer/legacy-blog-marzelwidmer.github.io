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


# Table of contents

* [Create Project](#CreateProject)
* [Install Jenkins](#InstallJenkins)
* [Add Edit Role To ServiceAccount Jenkins](#AddEditRoleToServiceAccountJenkins)
* [Add Role To Group](#AddRoleToGroup)
* [Deploy Application](#DeployApplication)
* [Development Environment Deployment ](#DevelopmentEnvironmentDeployment)
* [Test API](#TestAPI)
* [Testing Environment Deployment](#TestingEnvironmentDeployment)
* [Production Environment Deployment](#ProductionEnvironmentDeployment)
* [Jenkins Pipeline](#Jenkins Pipeline)


## Create Project  <a name="CreateProject"></a>
We are going to use the CLI to create some projects. 
You can, of course, use the [console](https://console.c3smonkey.ch:8443/console/catalog){:target="_blank"} if you prefer. 
Let's create our projects first:
```
oc login  
oc new-project development --display-name="Development Environment"
oc new-project testing --display-name="Testing Environment"    
oc new-project production --display-name="Production Environment"    
oc new-project jenkins --display-name="Jenkins CI/CD"  
```

## Install Jenkins   <a name="InstallJenkins"></a>
Create a Jenkins in the `Jenkins CI/CD` project with some storage. Take the Jenkins form the catalog and set some more memory and volume capacity on it.
Everything else we let the default values. 
After installation you can login with your Openshift account to the [Jenkins BlueOcean](https://jenkins-jenkins.apps.c3smonkey.ch/blue/organizations/jenkins/)  
![Jenkins-From-Catalog-1](/assets/img/2019/openshift-pipeline/Jenkins-from-catalog-1.png)
![Jenkins-From-Catalog-2](/assets/img/2019/openshift-pipeline/Jenkins-from-catalog-2.png)
![Jenkins-From-Catalog-3](/assets/img/2019/openshift-pipeline/Jenkins-from-catalog-3.png)

  
### Jenkins File From Source Repository
The pipeline [Jenkinsfile](https://raw.githubusercontent.com/marzelwidmer/catalog-service/master/Jenkinsfile){:target="_blank"} is provided in the source repository. 

## Add Edit Role To ServiceAccount Jenkins  <a name="AddEditRoleToServiceAccountJenkins"></a>
Let’s add in RBAC to our projects to allow the different service accounts to build, pro‐ mote, and tag images.
First we will allow the cicd project’s Jenkins service account edit access to all of our projects:
```
oc policy add-role-to-user edit system:serviceaccount:jenkins:jenkins -n development
oc policy add-role-to-user edit system:serviceaccount:jenkins:jenkins -n testing
oc policy add-role-to-user edit system:serviceaccount:jenkins:jenkins -n production
```

## Add Role To Group <a name="AddRoleToGroup"></a>
That we can pull our image from `testing` and `production` environment from the `development` registry. This are needed for pulling the Images across the projects.
```
oc policy add-role-to-group system:image-puller system:serviceaccounts:testing  \
        -n development
oc policy add-role-to-group system:image-puller system:serviceaccounts:production \
        -n development
```

# Deploy Application <a name="DeployApplication"></a>
Let's deploy first the application with the [S2i strategy](https://docs.openshift.com/container-platform/3.11/creating_images/s2i.html){:target="_blank"}.
before we will create the delivery pipeline.
## Development Environment Deployment  <a name="DevelopmentEnvironmentDeployment"></a> 
Let's change first the project to to development with the `oc project development` command.
```
oc project development

Now using project "development" on server "https://console.c3smonkey.ch:8443".```

Creat a new app with `oc new-app`.
```

We will use here the [fabric8/s2i-java](https://hub.docker.com/r/fabric8/s2i-java){:target="_blank"} to deploy our application and will point it to the master branch with the commant `oc new-app`
We also want expose the service `oc expose svc/catalog-service` to get a URL with the command `oc get route catalog-service` we will see the URL on the terminal. 

``` 
oc new-app fabric8/s2i-java:latest-java11~https://github.com/marzelwidmer/catalog-service.git#master; oc expose svc/catalog-service; oc get route catalog-service

--> Found Docker image 6414174 (7 weeks old) from Docker Hub for "fabric8/s2i-java:latest-java11"

    Java Applications
    -----------------
    Platform for building and running plain Java applications (fat-jar and flat classpath)

    Tags: builder, java

    * An image stream tag will be created as "s2i-java:latest-java11" that will track the source image
    * A source build using source code from https://github.com/marzelwidmer/catalog-service.git#master will be created
      * The resulting image will be pushed to image stream tag "catalog-service:latest"
      * Every time "s2i-java:latest-java11" changes a new build will be triggered
    * This image will be deployed in deployment config "catalog-service"
    * Ports 8080/tcp, 8778/tcp, 9779/tcp will be load balanced by service "catalog-service"
      * Other containers can access this service through the hostname "catalog-service"

--> Creating resources ...
    imagestream.image.openshift.io "s2i-java" created
    imagestream.image.openshift.io "catalog-service" created
    buildconfig.build.openshift.io "catalog-service" created
    deploymentconfig.apps.openshift.io "catalog-service" created
    service "catalog-service" created
--> Success
    Build scheduled, use 'oc logs -f bc/catalog-service' to track its progress.
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/catalog-service'
    Run 'oc status' to view your app.
route.route.openshift.io/catalog-service exposed
NAME              HOST/PORT                                       PATH   SERVICES          PORT       TERMINATION   WILDCARD
catalog-service   catalog-service-development.apps.c3smonkey.ch          catalog-service   8080-tcp                 None
```

Now take a look in the OpenShift [console](https://console.c3smonkey.ch:8443/){:target="_blank"} project `development`

![catalog-service-dev-deployment](/assets/img/2019/openshift-pipeline/catalog-service-dev-deployment.png)

Let's take a look what the S2i crated for us. 
This can be done with the following command `oc get all -n development --selector app=catalog-service`.

``` 
oc get all -n development --selector app=catalog-service                                  
NAME                          READY   STATUS    RESTARTS   AGE
pod/catalog-service-1-2rv5r   1/1     Running   0          56m

NAME                                      DESIRED   CURRENT   READY   AGE
replicationcontroller/catalog-service-1   1         1         1       56m

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/catalog-service   ClusterIP   172.30.216.240   <none>        8080/TCP,8778/TCP,9779/TCP   57m

NAME                                                 REVISION   DESIRED   CURRENT   TRIGGERED BY
deploymentconfig.apps.openshift.io/catalog-service   1          1         1         config,image(catalog-service:latest)

NAME                                             TYPE     FROM         LATEST
buildconfig.build.openshift.io/catalog-service   Source   Git@master   1

NAME                                         TYPE     FROM          STATUS     STARTED             DURATION
build.build.openshift.io/catalog-service-1   Source   Git@b49dff4   Complete   About an hour ago   1m10s

NAME                                             DOCKER REPO                                                    TAGS            UPDATED
imagestream.image.openshift.io/catalog-service   docker-registry.default.svc:5000/development/catalog-service   latest          About an hour ago
imagestream.image.openshift.io/s2i-java          docker-registry.default.svc:5000/development/s2i-java          latest-java11   About an hour ago

NAME                                       HOST/PORT                                       PATH   SERVICES          PORT       TERMINATION   WILDCARD
route.route.openshift.io/catalog-service   catalog-service-development.apps.c3smonkey.ch          catalog-service   8080-tcp                 None
~ 🐠
```


## Test API on Development <a name="TestAPI"></a>
Now let's test the amazing `/api/v1/animals/rando` API from `catalog-service` by hitting the following Rest endpoint 50 times. 
in a bash shell with the following command `for x in (seq 50); http "http://catalog-service-development.apps.c3smonkey.ch/api/v1/animals/random"; end`
```bash
for x in (seq 50); \
     http "http://catalog-service-development.apps.c3smonkey.ch/api/v1/animals/random"; \
end                                                                               

HTTP/1.1 200 OK
Cache-control: private
Content-Length: 14
Content-Type: text/plain;charset=UTF-8
Set-Cookie: 1e5e1500c4996e7978ef9efb67d863a1=1e12d12873c24c5c17782f3da537ed6a; path=/; HttpOnly

Burrowing Frog

HTTP/1.1 200 OK
Cache-control: private
Content-Length: 14
Content-Type: text/plain;charset=UTF-8
Set-Cookie: 1e5e1500c4996e7978ef9efb67d863a1=1e12d12873c24c5c17782f3da537ed6a; path=/; HttpOnly

Dogo Argentino

HTTP/1.1 200 OK
Cache-control: private
Content-Length: 7
Content-Type: text/plain;charset=UTF-8
Set-Cookie: 1e5e1500c4996e7978ef9efb67d863a1=1e12d12873c24c5c17782f3da537ed6a; path=/; HttpOnly
```




 
## Testing Environment Deployment <a name="TestingEnvironmentTesting"></a>
Let's change first to the testing project with `oc project testing`. 
We remember tha we have in our setup only one docker registry from this registry we want promote our Docker images to other projects in our OpenShift Cluster setup.
The access is now available because we dit the [Add Role To Group](#AddRoleToGroup). Now let's take a look at the ImageStream in the project `development` with the
following command `oc get is -n development` we will get the docker registry we need to create the a deployment configuration in the project `testing`. we are searching for the 
`catalog-service` docker registry.
``` 
oc get is -n development
NAME              DOCKER REPO                                                    TAGS            UPDATED
catalog-service   docker-registry.default.svc:5000/development/catalog-service   latest          About an hour ago
s2i-java          docker-registry.default.svc:5000/development/s2i-java          latest-java11   About an hour ago
```
 
Now let's create a deployment configuration for the `promoteQA` tag with `oc create dc catalog-service --image=docker-registry.default.svc:5000/development/catalog-service:promoteQA` 
Our Jenkins pipeline is configured to interact with this tag to promote between the environments. 
``` 
oc create dc catalog-service --image=docker-registry.default.svc:5000/development/catalog-service:promoteQA
deploymentconfig.apps.openshift.io/catalog-service created
```

Now we have to expose the 'dc/catalog-service' with the port `8080` who our Spring Boot App is running on.  
This easy done with the `oc expose dc catalog-service --port=8080` command.
``` 
oc expose dc catalog-service --port=8080
service/catalog-service exposed
```

Now also expose the service and get the route `oc expose service catalog-service --name=catalog-service; oc get route`
``` 
oc expose service catalog-service --name=catalog-service; oc get route
route.route.openshift.io/catalog-service exposed
NAME              HOST/PORT                                   PATH   SERVICES          PORT   TERMINATION   WILDCARD
catalog-service   catalog-service-testing.apps.c3smonkey.ch          catalog-service   8080                 None
```

We have to patch the `dc` `imagePullPolicy` form `IfNotPresent` to `Always` with `oc patch` command. The default is set to `IfNotPresent`, but we wish to always trigger a deployment when we tag a new image
``` 
 oc patch dc/catalog-service  -p \
      '{"spec":{"template":{"spec":{"containers":[{"name":"default-container","imagePullPolicy":"Always"}]}}}}'
```


## Production Environment Deployment <a name="ProductionEnvironmentTesting"></a>
The same we did on the [Testing Environment Deployment ](#TestingEnvironmentDeployment) we will make now on the production environment. 
But we configure the promotion tag `promotePRD` in the deployment configuration.

``` 
oc project production
oc create dc catalog-service --image=docker-registry.default.svc:5000/development/catalog-service:promotePRD
oc patch dc/catalog-service  -p \
     '{"spec":{"template":{"spec":{"containers":[{"name":"default-container","imagePullPolicy":"Always"}]}}}}'
oc expose dc catalog-service --port=8080
oc expose svc/catalog-service
```



## Jenkins Pipeline  <a name="Jenkins Pipeline"></a>

So now let's create a `BuildConfig` for the `catalaog-service` with the following [catalog-service-jenkins-pipeline](/assets/img/2019/openshift-pipeline/catalog-service-pipeline.yaml){:target="_blank"} configuration
in the Jenkins namespace (project) let's do it with `oc create -n jenkins -f https://blog.marcelwidmer.org/assets/img/2019/openshift-pipeline/catalog-service-pipeline.yaml`

``` 
oc create -n jenkins -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-pipeline/catalog-service-pipeline.yaml
buildconfig.build.openshift.io/catalog-service-pipeline created
```

When you go now in the OpenShift [console](https://console.c3smonkey.ch:8443/console/project/jenkins/browse/pipelines){:target="_blank"} in the project `Jenkins` in the section.
`Builds/Pipelines` you will something like this.
![Catalog Service Pipeline](/assets/img/2019/openshift-pipeline/catalog-service-pipeline-created.png)


### Run Jenkins Pipeline 
Now is time to run the pipeline with the command `oc start-build catalog-service-pipeline -n jenkins` 

```
oc start-build catalog-service-pipeline -n jenkins
build.build.openshift.io/catalog-service-pipeline-1 started
```

After a while you will see something like this. For production deployment we configured our pipeline with a approvable step.

![Catalog Service Pipeline approvable](/assets/img/2019/openshift-pipeline/catalog-service-pipeline-approvable.png)

Now is time to approve the application and hit the 
After the approve button in the pipeline to deploy to the production namespace.

![Catalog Service Pipeline success](/assets/img/2019/openshift-pipeline/catalog-service-pipeline-success.png)



> **_References:_**  
>   [https://blog.openshift.com/decrease-maven-build-times-openshift-pipelines-using-persistent-volume-claim/](https://blog.openshift.com/decrease-maven-build-times-openshift-pipelines-using-persistent-volume-claim/) 
>   [https://github.com/redhat-cop/container-pipelines/tree/master/basic-spring-boot](https://github.com/redhat-cop/container-pipelines/tree/master/basic-spring-boot)   


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
