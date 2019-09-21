---
layout: post
title: Spring Boot meets k8s
date: 2019-09-18
last_modified: 2019-09-21
description: Integrate Spring Boot with Kubernetes ConfigMap # Add post description (optional)
img: 2019/spring-boot-k8s/springBoot-k8s.png  # Add image post (optional)
tags: [Blog, k8s, OpenShift, OKD, Spring Boot]
author: # Add name author (optional)
--- 

# Table of contents
* [Maven](#MavenConfiguration)
* [Application Configuration](#ApplicationConfiguration)
* [Configured Service Account - RBAC policy](#RBACpolicy)
* [Deploy ConfigMap](#DeployConfigMap)
 
Now is time to configure our [microservices](https://github.com/marzelwidmer/microservices-demo){:target="_blank"} to send the tracing 
logs to [Jaeger](http://blog.marcelwidmer.org/jaeger/). The configuration `opentracing.jaeger.http-sender.url` in configuration `application.yaml` file looks like below in the sources.
```yaml
opentracing:
  jaeger:
    log-spans: true
    http-sender:
      url: http://localhost:14268/api/traces
``` 

The `opentracing.jaeger.http-sender.url` we are looking for we get form the section [Get Route Host in the Jaeger post](http://blog.marcelwidmer.org/jaeger/#GetRouteHost)
We will use the `ConfigMap` approach with the [Spring Cloud Kubernetes](https://spring.io/projects/spring-cloud-kubernetes){:target="_blank"} starters.

## Maven <a name="MavenConfiguration"></a>
Update Maven Configuration with [Spring Cloud Kubernetes](https://cloud.spring.io/spring-cloud-static/spring-cloud-kubernetes/1.0.3.RELEASE/single/spring-cloud-kubernetes.html){:target="_blank"} library.

### Dependency Management `spring-cloud-dependencies`
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Spring Cloud Version `spring-cloud.version` 
```xml
<properties>
    ...
    <spring-cloud.version>Greenwich.SR3</spring-cloud.version>
</properties>
```

### Dependency `spring-cloud-starter-kubernetes-config` 
```xml
<!-- Kubernetes -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-config</artifactId>
</dependency>
```

## Application Configuration <a name="ApplicationConfiguration"></a>
The application may need to detect changes on external property sources and update their internal status to reflect the new configuration. 
The reload feature of `Spring Cloud Kubernetes` is able to trigger an application reload when a related `ConfigMap` changes.

This feature is disabled by default and can be enabled using the configuration property `spring.cloud.kubernetes.reload.enabled=true` 
 in the `application.yaml` file.

The configuration `spring.cloud.kubernetes.reload.strategy=restart_context` will restart the whole Spring ApplicationContext gracefully.
               
```yaml
spring:
  cloud:
    kubernetes:
      reload:
        enabled: true
        strategy: restart_context
```
Configure the `management.endpoint.restart.enabled=true`
```bash
management:
  endpoint:
    restart:
      enabled: true
```

## Configured Service Account - RBAC policy <a name="RBACpolicy"></a>
To read the `ConfigMap` we have to give to the service account in the default namespace access right. This can be done to give just 
`view` access `oc policy add-role-to-user view system:serviceaccount:development:default` 
 
The better solution is [configure ClusterRole](#ConfigureClusterRole) 

> ⚠️ **Avoid no RBAC policy match exception**:  
                                        .fabric8.kubernetes.client.KubernetesClientException: 
                                        Failure executing: GET at: https://172.30.0.1/api/v1/namespaces/development/pods/order-service-35-wj25f. 
                                            Message: Forbidden!Configured service account doesnt have access. 
                                            Service account may have been revoked. pods "order-service-35-wj25f" is 
                                                forbidden: User "system:serviceaccount:development:default" cannot get pods in the namespace "development": no RBAC policy matched.
                                         

# Create ClusterRole  <a name="ConfigureClusterRole"></a>
Additional you can also create a `ClusterRole` for Spring components let it named `spring-roles`.
Create a file `service-account-for-spring-cloud-k8s-access.yaml`
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: spring-roles
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods","configmaps"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: allow-spring-to-access-cluster
subjects:
  - kind: ServiceAccount
    name: default
    namespace: default
roleRef:
  kind: ClusterRole
  name: spring-roles
  apiGroup: rbac.authorization.k8s.io
```

Login in with privileged user `oc login -u <privileged user>`.
```bash
$ oc apply -f service-account-for-spring-cloud-k8s-access.yaml
```

Now when you check che Cluster Console under _Administration/Roles_ and you search for `spring` you will find the role. 

![cluster-console-search-spring-roles.png](/assets/img/2019/spring-boot-k8s/cluster-console-search-spring-roles.png)
![cluster-console-spring-roles.png](/assets/img/2019/spring-boot-k8s/cluster-console-spring-roles.png)




## Deploy ConfigMap <a name="DeployConfigMap"></a>
Now is time to create our `ConfigMap` and `apply` it in the `development` namespace for the `oder-service`
You can do it directly in a shell.
```bash
$ echo "apiVersion: v1
kind: ConfigMap
metadata:
    #  matches the spring app name as defined in application.yml
    name: order-service
data:
    #  must be named 'application.yaml' or be the only key in this config
    #  refer to Spring Cloud Kubernetes Config documentation or source code
    application.yaml: |
        opentracing:
            jaeger:
                http-sender:
                    url: http://jaeger-collector-jaeger.apps.c3smonkey.ch/api/traces" | oc apply -f -
```

Better is you create a `ConfigMap` file and the you use the `apply` command.  
```bash
$ oc apply -f deployments/configmap.yaml
```

When you hit the service again you will see some traces in the Jaeger now.
```bash
$ for x in (seq 50); http "http://order-service-development.apps.c3smonkey.ch/api/v1/orders/random"; end
```

![Jaeger-Order-Service-Traces](/assets/img/2019/spring-boot-k8s/Jaeger-Order-Service-Traces.png)








### Addition Commands
Create `ConfigMap` from file is also a useful way to create a `ConfigMap`
```bash
$ oc create configmap order-service --from-file=src/main/resources/application.yaml
```

Get all `ConfigMaps`
```bash
$ oc get configmaps
```

Get `ConfigMap` as `yaml`
```bash
$ oc get configmap order-service -o yaml
```

Describe `ConfigMap`
```bash
$ oc describe configmap order-service
```

Delete `ConfigMap`
```bash
$ oc delete configmap order-service
```

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/