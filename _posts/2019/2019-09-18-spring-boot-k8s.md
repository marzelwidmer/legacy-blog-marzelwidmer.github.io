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


 




`ConfigMap` in our kubernetes aka k8s / OpenShift environment to override the `Jaeger` 





# Kubernetes ConfigMap Support

## Add Maven `spring-cloud-dependencies` dependency management  
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

## Add `spring-cloud.version` properties.
```xml
<properties>
		...
		<spring-cloud.version>Greenwich.SR3</spring-cloud.version>
	</properties>
```

## Add Maven dependency `spring-cloud-starter-kubernetes-config` 
```xml
		<!-- Kubernetes -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-kubernetes-config</artifactId>
		</dependency>
```

## Update RBAC policy
```bash
$ oc policy add-role-to-user view system:serviceaccount:development:default
```

To avoid the following exception.
```bash
.fabric8.kubernetes.client.KubernetesClientException: 
Failure executing: GET at: https://172.30.0.1/api/v1/namespaces/development/pods/order-service-35-wj25f. 
    Message: Forbidden!Configured service account doesn't have access. 
    Service account may have been revoked. pods "order-service-35-wj25f" is 
        forbidden: User "system:serviceaccount:development:default" cannot get pods in the namespace "development": no RBAC policy matched.
```


# ConfigMap
Apply `ConfigMap`
```bash
$ oc apply -f deployments/configmap.yaml
```

## Additional ConfigMap Commands
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

# Wat running POD
```bash
$ watch oc get pods --field-selector=status.phase=Running                                                                         28.6m  Thu Sep 19 16:14:40 2019
```

# Tail logfile
```bash
$ oc logs -f order-service-37-hh2tb
```
 



[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/