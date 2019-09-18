---
layout: post
title: OpenShift - Semantic Release with Jenkins Maven Fabric8 Delivery Pipeline
date: 2019-09-08
description: # Add post description (optional)
img: 2019/openshift-semantic-release-with-jenkins-maven-fabric8-delivery-pipeline/out-0010.png  # Add image post (optional)
tags: [Blog, Jenkins, OpenShift, k8s, OKD, CI/CD, Docker, Images, Container, Release, Versioning, Tagging, Semantic Release, Semantic Versioning, fabric8]
author: # Add name author (optional)
---

# Table of contents
* [Setup Deployment](#SetupDeployment)
* [Jenkins Pipeline](#JenkinsPipeline)
* [WebHooks](#WebHooks)
* [Private Repository](#privateRepo)
* [Semantic Release Jenkins Pipeline](#SemanticReleaseJenkinsPipeline)
 

## Setup Deployment <a name="SetupDeployment"></a>
```bash
$ oc new-project development --display-nam  e="Development Environment"
```

Deploy application with the `maven.fabric8.io` plugin in  `development` stage from local machine.
```bash
$ ./mvnw fabric8:deploy -Dfabric8.namespace=development
$ oc policy add-role-to-user edit system:serviceaccount:jenkins:jenkins -n development
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
$ oc create dc customer-service --image=docker-registry.default.svc:5000/development/customer-service:promotePROD -n production
$ oc patch dc/customer-service  -p \
     '{"spec":{"template":{"spec":{"containers":[{"name":"default-container","imagePullPolicy":"Always"}]}}}}' -n production
$ oc expose dc customer-service -n production --port=8080
$ oc expose svc/customer-service -n production
```

## Jenkins Pipeline  <a name="JenkinsPipeline"></a>
Let's creat a Jenkins Pipeline for the `customer-service` in the project `jenkins`.
```bash
$ oc create -n jenkins -f \
    https://blog.marcelwidmer.org/assets/img/2019/openshift-semantic-release-with-jenkins-maven-fabric8-delivery-pipeline/customer-service-pipeline.yaml
```

## WebHooks <a name="WebHooks"></a>
How we can create a [GitHub WebHook](https://github.com/marzelwidmer/customer-service/settings/hooks){:target="_blank"} for a public Git repository take a look at the following post there we created already a  
[WebHook](http://blog.marcelwidmer.org/openshift-delivey-pipeline/#WebHooks){:target="_blank"} for the `catalog-service` but here some `oc` commands
for the `customer-service`.
```bash
$ oc set triggers bc/customer-service-pipeline --from-github  -n jenkins 
```
Grab the `Secret`.
```bash
$ oc get bc/customer-service-pipeline -n jenkins -o json | jq '.spec.triggers[].github.secret'
```
Grab `Webhook GitHub URL`. 
```bash
$ oc describe bc/customer-service-pipeline -n jenkins
```

## Private Repository access with secrets <a name="privateRepo"></a>
Create a `generic secret` link this secret with the `builder`.
Annotate and label it for the Jenkins sync PlugIn. And finally update the `bc/customer-service-pipeline` with this secret.
First you have to create an `AccessToken` in your [GitHub Tokens Settings](https://github.com/settings/tokens){:target="_blank"} let it named like `openshift-source-builder`
add `repo` and `user` access because this token will be used for `Semantic Release`
> ⚠️ **GitHub Token**: Create a AccessToken with `repo,user` rights under your [GitHub Tokens Settings](https://github.com/settings/tokens) documentation.
 
![openshift-source-builder-github-token](/assets/img/2019/openshift-semantic-release-with-jenkins-maven-fabric8-delivery-pipeline/openshift-source-builder-github-token.png)

```bash
$ oc create secret generic ci-user-at-github \
      --from-literal=username=machineuser \
      --from-literal=password=<accesstoken> \
      --type=kubernetes.io/basic-auth \
      -n jenkins
$ oc secrets link builder ci-user-at-github \
    -n jenkins
$ oc annotate secret/ci-user-at-github \
      'build.openshift.io/source-secret-match-uri-1=https://github.com/marzelwidmer/*' \
    -n jenkins
$ oc label secret ci-user-at-github credential.sync.jenkins.openshift.io=true \
    -n jenkins
$ oc set build-secret bc/customer-service-pipeline ci-user-at-github --source
```

When you check now the Jenkins you will see the `ci-user-at-github` under credentials `https://<jenkins-url>/credentials/` 

![sync.jenkins](/assets/img/2019/openshift-semantic-release-with-jenkins-maven-fabric8-delivery-pipeline/sync.jenkins.openshift.io.png)

You will also find the a secret `ci-user-at-github` in the `jenkins` project in the OpenShift console.

![secret-ci-user-at-github](/assets/img/2019/openshift-semantic-release-with-jenkins-maven-fabric8-delivery-pipeline/secret-ci-user-at-github.png)


## Semantic Release Jenkins Pipeline <a name="SemanticReleaseJenkinsPipeline"></a>
First I want say the inspiration I get from the [semantic-release](https://github.com/semantic-release/semantic-release){:target="_blank"} automates the whole package 
release workflow including: determining the next version number, generating the release notes and publishing the package.

> 😎 This removes the immediate connection between human emotions and version numbers, strictly following the Semantic Versioning specification.

In the case I don't found any Maven PlugIn who works in my setup _out-of-the-box_ and I am running here in a `Maven Slave` and don't want create a` Maven-Node Slave`
I chose to follow a setup with just Git commands and a combination with the [jgitver-maven-plugin](https://github.com/jgitver/jgitver-maven-plugin){:target="_blank"}.

 ![okd-customer-service-pipeline](/assets/img/2019/openshift-semantic-release-with-jenkins-maven-fabric8-delivery-pipeline/okd-customer-service-pipeline.png)

After pushing some code in the `customer-service` repository the Jenkins pipeline start run. 
It will be tag the source repository if needed based on the commit message inspired on the follow [commit message format](https://github.com/semantic-release/semantic-release#commit-message-format).
> ⚠️ **Commit Message Format**: Current version 1.0.0 will change the version like:
>
>Major version (2.0.0) 👉🏼 breaking:|major:|BREAKING CHANGE:
>
>Minor version (1.1.0) 👉🏼 feature:|minor:|feat:
>
>Patch version (1.0.1) 👉🏼 fix:|patch:|docs:|style:|refactor:|perf:|test:|chore:

It will also create the image tags if needed.

![okd-customer-service-image-tags](/assets/img/2019/openshift-semantic-release-with-jenkins-maven-fabric8-delivery-pipeline/okd-customer-service-image-tags.png)

Take also a look at the [Jenkins BlueOcean](https://jenkins.io/projects/blueocean/){:target="_blank"} pipeline. 

![blueocean-customer-service-pipeline](/assets/img/2019/openshift-semantic-release-with-jenkins-maven-fabric8-delivery-pipeline/blueocean-customer-service-pipeline.png)

Or at the deployed _customer-service_.

[Customer Service Swagger](http://customer-service-production.apps.c3smonkey.ch/swagger-ui.html){:target="_blank"}
![customer-service-swagger.png](/assets/img/2019/openshift-semantic-release-with-jenkins-maven-fabric8-delivery-pipeline/customer-service-swagger.png)


The _changelog_ you can find under [Changelog](https://jenkins-jenkins.apps.c3smonkey.ch/job/jenkins/job/jenkins-customer-service-pipeline/lastSuccessfulBuild/artifact/target/changelog.html){:target="_blank"}
 

I know the above pipeline is a bit chatty because of this I explain my steps here a bit compromised in some pipeline steps. 
Ok let's create a  _Jenkinsfile_ in your project repository I prefer to put the _Jenkinsfile_ in a folder _jenkins_.

First we configure a _Maven Slave_ for this we configure the _agent node_ with the laben `maven` because our porject is a Maven project.
```groovy
pipeline {
    // Agent Maven
    agent {
        node {
            label 'maven'
        }
    }
}
```

Now lets define some _environment_ variables we can parameterize the pipeline also for other projects later.
```groovy
    // Environment
    environment {
        REPOSITORY = "github.com/marzelwidmer/customer-service.git"
        BRANCHES = "[[name: '*/master']]"
        GIT_TAG_MESSAGE = 'ci-release-bot'
        GIT_TAG_USER_EMAIL =  "jenkins@c3smonkey.ch"
        GIT_TAG_USER_NAME = "Jenkins"
        DEV_ENVIRONMENT = 'development'
        TEST_ENVIRONMENT = 'testing'
        PROD_ENVIRONMENT = 'production'
        APP_NAME = 'customer-service'
    }
```

(1) The following step will first check the last _GIT_COMMIT ID_ and download a _ci-semver.sh_ script who is hosted in a other central repository who can used for central `CI/CD` scripts across different 
project Git repositories. 

(2) This shell script will then check the Git history if there a commit message who match our Semantic Version naming convention to compute the next version.

(3) The next command will then call the _Maven_ build and test lifecycle and use the [jgitver](https://github.com/jgitver/jgitver){:target="_blank"} plugin to pass the version we get before from the 
_ci-semver.sh_ script.  

> 💡 ** jgitver **: [https://www.youtube.com/watch?v=mQmH_Ws9GFI](https://www.youtube.com/watch?v=mQmH_Ws9GFI){:target="_blank"}

(4) This section will store our _JUnit_ test results in the Jenkins.

(5) Tag Git repository if needed.

```groovy
    // Stages
    stages {
        // Setup
        stage('build and test application') {
            steps {
                script {
                    // Last Git commit
                    LAST_GIT_COMMIT = sh(
                            script: 'git --no-pager show -s --format=\'%Cblue %h %Creset %s %Cgreen %an %Creset (%ae)\'',
                            returnStdout: true
                    ).trim()
                    echo "Last Git commit: ${LAST_GIT_COMMIT}"
                    // Download script  (1)
                    sh '''
                        echo download ci-semver.sh script from remote repository
                        curl https://raw.githubusercontent.com/marzelwidmer/git-semantic-commits/master/ci-semver.sh  --output ./jenkins/ci-semver.sh
                        chmod +x ./jenkins/ci-semver.sh
                    '''
                    // Compute next version (2)
                    NEXT_VERSION = sh(
                            script: "./jenkins/ci-semver.sh",
                            returnStdout: true
                    ).trim()
                    echo "NEXT_VERSION : ${NEXT_VERSION}"
                    withCredentials([usernamePassword(credentialsId: 'jenkins-ci-user-at-github', usernameVariable: 'USERNAME', passwordVariable: 'TOKEN')]) {
                        git(branches: "$BRANCHES", changelog: true, url: "https://$TOKEN:x-oauth-basic@$REPOSITORY")
                    }
                    // Maven build and test (3)
                    sh("""
                        echo next version will be $NEXT_VERSION
                        ./mvnw validate
                        ./mvnw validate -Djgitver.use-version=$NEXT_VERSION
                        ./mvnw package -DskipTests -Djgitver.use-version=$NEXT_VERSION
                        echo "run tests"
                        sh './mvnw test'
                    """)
                    // Store Junit results (4)
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                    // Tag Git Repository (5)
                    echo "Create tag with version: ${NEXT_VERSION}"
                    sh("""
                        echo set git config for tagging
                        git config --global user.email '${GIT_TAG_USER_EMAIL}'
                        git config --global user.name '${GIT_TAG_USER_NAME}'

                        if [ \$(git tag -l $NEXT_VERSION) ]; then
                            echo tag exist already
                        else    
                            git tag -a -m '${GIT_TAG_MESSAGE}' ${NEXT_VERSION}
                            git push --follow-tags
                        fi
                    """)
                }
            }
        }
}
```



> **_References:_**  
>[Jenkins Client Plugin](https://github.com/openshift/jenkins-client-plugin)
>[Best Practices for Managing Docker Versions](https://www.youtube.com/watch?v=MqsG9-HEcTw) 
>[CI/CD - A/B - OpenShift - Jenkins](https://dzone.com/articles/continuous-delivery-with-openshift-and-jenkins-ab)
>[GitLab Private Repository](https://cookbook.openshift.org/building-and-deploying-from-source/how-can-i-build-from-a-private-repository-on-gitlab.html)
>[Personal Access Tokens - Private Repository](https://blog.openshift.com/private-git-repositories-part-3-personal-access-tokens/)
>[Pipelines with git tags](https://jenkins.io/blog/2018/05/16/pipelines-with-git-tags/)
>[jgitver - goal](https://jgitver.github.io/#_goal)
>[jgitver-maven-plugin](https://github.com/jgitver/jgitver-maven-plugin)   
>[Semantic commits for git](https://codito.in/semantic-commits-for-git)
>[git-semantic-commits](https://github.com/marzelwidmer/git-semantic-commits)
>[Jenkins - semantic-versioning-plugin](https://wiki.jenkins.io/display/JENKINS/semantic-versioning-plugin)
>[Jenkins - Git+Parameter+Plugin](https://wiki.jenkins.io/display/JENKINS/Git+Parameter+Plugin)
>[wilsonmar - jenkins2-pipeline](https://wilsonmar.github.io/jenkins2-pipeline/)
>[Get Jenkins GDSL working with IntelliJ IDEA](https://gist.github.com/arehmandev/736daba40a3e1ef1fbe939c6674d7da8)


