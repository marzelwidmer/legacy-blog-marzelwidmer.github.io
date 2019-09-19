---
layout: post
title: Spring Boot meets k8s
date: 2019-09-08
last_modified: 2019-09-19
description: Integrate Spring Boot with Kubernetes # Add post description (optional)
img: 2019/spring-boot-k8s/springBoot-k8s.png  # Add image post (optional)
tags: [Blog, k8s, OpenShift, OKD, Spring Boot]
author: # Add name author (optional)
--- 

Now is time to configure our [microservices](https://github.com/marzelwidmer/microservices-demo){:target="_blank"} to send the tracing logs to [Jaeger](http://blog.marcelwidmer.org/jaeger/).
The configuration `opentracing.jaeger.http-sender.url` in configuration `application.yaml` file looks like below in the sources.
```yaml
opentracing:
  jaeger:
    log-spans: true
    http-sender:
      url: http://localhost:14268/api/traces
``` 

The `opentracing.jaeger.http-sender.url` we are looking for we get form the section [Get Route Host in the Jaeger post](http://blog.marcelwidmer.org/jaeger/#GetRouteHost)
We will use the `ConfigMap` approach with the [Spring Cloud Kubernetes](https://spring.io/projects/spring-cloud-kubernetes){:target="_blank"} starters.

## Update Maven Configuration
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

## application.yaml
For reloading the Spring context after update the `ConfigMap` is important that there is the following  
configuration `spring.cloud.kubernetes.reload.enabled=true` `spring.cloud.kubernetes.reload.strategy=restart_context` and 
`management.endpoint.restart.enabled=true` in our `application.yaml` configured. 
               
```yaml
spring:
  cloud:
    kubernetes:
      reload:
        enabled: true
        strategy: restart_context

management:
  endpoint:
    restart:
      enabled: true
```

## Update RBAC policy
In OpenShift that we can read from the `ConfigMap` we have to update the RBAC policy
```bash
$ oc policy add-role-to-user view system:serviceaccount:development:default
```

to avoid the following exception
```bash
.fabric8.kubernetes.client.KubernetesClientException: 
Failure executing: GET at: https://172.30.0.1/api/v1/namespaces/development/pods/order-service-35-wj25f. 
    Message: Forbidden!Configured service account doesn't have access. 
    Service account may have been revoked. pods "order-service-35-wj25f" is 
        forbidden: User "system:serviceaccount:development:default" cannot get pods in the namespace "development": no RBAC policy matched.
```


## Deploy ConfigMap
With the following command you can deploy the `ConfigMap` in the `development` namespace.
```bash
echo "apiVersion: v1
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

Or you just use the _apply_ `ConfigMap` following command when you save the configuration in a file.
```bash
$ oc apply -f deployments/configmap.yaml
```

This are also some useful commands

Create `ConfigMap` from file.
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

Watch running POD
```bash
$ watch oc get pods --field-selector=status.phase=Running                                                                         28.6m  Thu Sep 19 16:14:40 2019
```

Tail logfile
```bash
$ oc logs -f order-service-37-hh2tb
```
 



[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/