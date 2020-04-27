--- 
layout: post
title: Create Kotlin Project with Spring Initializr and HTTPie
description: Create Maven Spring Project with HTTPie from start.spring.io - Spring Boot - Maven
date:   2020-04-12 10:05:55 +0300
image: IMG_0079.jpeg
tags: [Spring Boot, Kotlin]
---
                                                                                                                
# Create Kotlin Maven Project with HTTPie from start.spring.io 
Let's create and extract a `Maven` `Kotlin` project with some dependecies `actuator` `data-mongodb-reactive` `webflux` and `cloud-gateway`
The `https://start.spring.io`

```bash
http https://start.spring.io/starter.tgz \
    dependencies==actuator,data-mongodb-reactive,webflux,cloud-gateway \
    description=="Demo project Kotlin Sidecar Gateway" \
    applicationName==SidecarGatewayApplication \
    name==kotlin-sidecar-gateway \
    groupId==ch.keepcalm \
    artifactId==kotlin-sidecar-gateway \
    packageName==ch.keepcalm.demo \
    javaVersion==11 \
    language==kotlin \
    baseDir==kotlin-sidecar-gateway | tar -xzvf -
```

# Banner
Download Banner in the `src/main/resources` folder.
```bash
http https://raw.githubusercontent.com/marzelwidmer/marzelwidmer.github.io/master/img/2020/spring-initializr/banner.txt \
    > kotlin-sidecar-gateway/src/main/resources/banner.txt
```

# Spring Application Name
Configure `spring.applicatin.name`
```bash
echo "spring:
  application:
    name: kotlin-sidecar-gateway" | > kotlin-sidecar-gateway/src/main/resources/application.yaml
```
Remove `application.properties`
```bash
rm kotlin-sidecar-gateway/src/main/resources/application.properties
```



[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

