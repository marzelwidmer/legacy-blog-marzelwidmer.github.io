---
layout: post
title: Reactive Spring Boot with Resilience4j CircuitBreaker
date: 2020-04-18
last_modified: 2020-04-18
description: CircuitBreaker pattern with Kotlin Reactive Spring Boot Application
img: 2020/resilience4j-circuitbreaker/kotlin-spring-reactive.jpg
tags: [Spring Boot, Spring Reactive, Resilience4j CircuitBreaker, Kotlin]
author: # Add name author (optional)
--- 

# Reactive Spring Boot with Resilience4j CircuitBreaker

Let's create a Sample Application with Kotlin and Reactive Spring Boot with the [spring inializr](https://start.spring.io/) Rest Endpoint.
See also my other post [Spring Initializr and HTTPie](https://blog.marcelwidmer.org/spring-initializr/)


```bash
http https://start.spring.io/starter.tgz \
    dependencies==actuator,webflux,cloud-resilience4j \
    description=="Demo project Kotlin Spring Boot with Resilience4j" \
    applicationName==Resilience4jApplication \
    name==kboot-resilience4j \
    groupId==ch.keepcalm \
    artifactId==kboot-resilience4j \
    packageName==ch.keepcalm.demo \
    javaVersion==11 \
    language==kotlin \
    baseDir==kboot-resilience4j| tar -xzvf -

```





The example source code can be found here [GitHub](https://github.com/marzelwidmer/kboot-resilience4j)



> **_References:_**  
>[Resilience4j docs](https://resilience4j.readme.io/docs)


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
