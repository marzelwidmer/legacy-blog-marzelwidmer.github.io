---
layout: post
title: Create Kotlin Project with Spring Initializr and HTTPie
date: 2020-04-12
last_modified: 2020-04-12
description: Create Maven Spring Project with HTTPie from start.spring.io - Spring Boot - Maven   # Add post description (optional)
img: 2020/spring-initializr/initializr.png # Add image post (optional)
tags: [spring-initializr, Spring Boot, Kotlin]
author: # Add name author (optional)
--- 
                                                                                                                
# Create Kotlin Maven Project with HTTPie from start.spring.io 

The following command extract a Kotlin Maven Project generated from the `https://start.spring.io`

```bash
http https://start.spring.io/starter.tgz \
dependencies==actuator,data-mongodb-reactive,webflux,cloud-gateway \
description=="Demo project Kotlin Sidecar Gateway" \
applicationName==SidecarGatewayApplication name==kotlin-sidecar-gateway groupId==ch.keepcalm \
artifactId==kotlin-sidecar-gateway packageName==ch.keepcalm.demo \
javaVersion==11 language==kotlin \
baseDir==kotlin-sidecar-gateway | tar -xzvf -
```

# Download Banner
Download Banner in the `src/main/resources` folder.
```bash
http https://raw.githubusercontent.com/marzelwidmer/marzelwidmer.github.io/master/assets/img/2020/spring-initializr/banner.txt > kotlin-sidecar-gateway/src/main/resources/banner.txt
```


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

