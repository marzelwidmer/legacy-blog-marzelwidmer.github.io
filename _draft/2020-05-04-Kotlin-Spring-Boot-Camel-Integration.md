---
layout: post
title: Camel with Kotlin and SpringBoot
description: Camel with Kotlin and SpringBoot
date: 2020-05-04 12:05:55 +0300
last_modified: 2020-05-04 12:05:55 +0300
image: k8s.png
tags: [Spring Boot, Kotlin]
--- 
# Apache Camel with Kotlin and SpringBoot

## Create Project

Run the following commands :
```bash
export KBOOT_NAME=kboot-camel
export KBOOT_APPL_NAME=KbootCamel

http https://start.spring.io/starter.tgz \
    dependencies==actuator,webflux,camel \
    description=="Demo project Spring Boot" \
    applicationName==$KBOOT_APPL_NAME \
    name==$KBOOT_NAME \
    groupId==ch.keepcalm \
    artifactId==$KBOOT_NAME \
    packageName==ch.keepcalm.demo \
    javaVersion==11 \
    language==kotlin \
    bootVersion==2.3.0.RC1 \
    baseDir==$KBOOT_NAME| tar -xzvf -

cd $KBOOT_NAME

http https://raw.githubusercontent.com/marzelwidmer/marzelwidmer.github.io/master/img/banner.txt \
    > src/main/resources/banner.txt

rm src/main/resources/application.properties

echo "spring:
  application:
    name: $KBOOT_NAME" | > src/main/resources/application.yaml

idea .
```




> **_References:_**  
>[Spring Tips Apache Camel](https://spring.io/blog/2018/05/23/spring-tips-apache-camel)
 

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

