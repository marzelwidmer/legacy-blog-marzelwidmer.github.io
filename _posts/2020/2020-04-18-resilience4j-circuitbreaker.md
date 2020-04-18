---
layout: post
title: Reactive Spring Boot with Resilience4j CircuitBreaker
date: 2020-04-18
last_modified: 2020-04-18
description: Resilience4j - CircuitBreaker with Kotlin Reactive Spring Boot Application
img: 2020/resilience4j-circuitbreaker/kotlin-spring-reactive.jpg
tags: [Spring Boot, Spring Reactive, Resilience4j CircuitBreaker, Kotlin]
author: # Add name author (optional)
--- 

# Reactive Spring Boot with Resilience4j CircuitBreaker

## Create Project
Let's create a Sample Application with Kotlin and Reactive Spring Boot with the [spring inializr](https://start.spring.io/) Rest Endpoint. We will take the latest and greates Spring Boot version `2.3.0.M4` and language `kotlin` with the following dependencies:
* actuator
* webflux
* cloud-resilience4j

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
    bootVersion==2.3.0.M4 \
    baseDir==kboot-resilience4j| tar -xzvf -
```
Add Customer Banner
```bash
http https://raw.githubusercontent.com/marzelwidmer/marzelwidmer.github.io/master/assets/img/2020/spring-initializr/banner.txt \
    > kboot-resilience4j/src/main/resources/banner.txt
```
Configure `spring.applicatin.name`
```bash
echo "spring:
  application:
    name: kboot-resilience4j" | > kboot-resilience4j/src/main/resources/application.yaml
```
Remove `application.properties`
```bash
rm kboot-resilience4j/src/main/resources/application.properties
```

See also my other post [Spring Initializr and HTTPie](https://blog.marcelwidmer.org/spring-initializr/)

## Service 
Let's create a slow service `TurtleService` with a response class `TurtleServiceResponse`.
The Publisher have some nice functions to simulate a slow service. `.delayElement(Duration.ofSeconds(delayInSeconds.toLong()))`
We also will log the message with the `.doOnNext { log.info(it.message) }` function on server side.

```kotlin
@Service
class TurtleService {
    private val log = LoggerFactory.getLogger(javaClass)

    fun readySetGo(name: String?): Mono<TurtleServiceResponse> {
        name?.map {
            val delayInSeconds = (0..10).random()
            val msg = "$name Ready, set, go!! this call took $delayInSeconds"
            return Mono.just(TurtleServiceResponse(message = msg))
                .delayElement(Duration.ofSeconds(delayInSeconds.toLong()))
                .doOnNext { log.info(it.message) }
        }.isNullOrEmpty()
            .apply {
                return Mono.error(NullPointerException())
            }
    }
}
data class TurtleServiceResponse(val message: String)
```

## API
Let's create a REST API `/slow/{name}` with the [Reactive router Kotlin DSL](https://docs.spring.io/spring-framework/docs/current/kdoc-api/spring-framework/org.springframework.web.reactive.function.server/-router-function-dsl/index.html). 
A easy way to create a `WebFlux.fn` [RouterFunction](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/reactive/function/server/RouterFunctions.html)

We start with a `Hello World` Endpoint `/hello/{name}` and will refactor it later. 
```kotlin
fun main(args: Array<String>) {
    runApplication<Resilience4jApplication>(*args){
        addInitializers(
            beans { // this: BeanDefinitionDsl
                bean { // this: BeanDefinitionDsl.BeanSupplierContext
                    router { // this: RouterFunctionDsl
                        GET("/hello/{name}") { // it: ServerRequest
                            ok().body(fromValue("Hello ${it.pathVariable("name")}"))
                        }
                    }
                }
            }
        )
    }
}
```

`http :8080/hello/world`

```bash
HTTP/1.1 200 OK
Content-Length: 11
Content-Type: text/plain;charset=UTF-8

Hello world
```











The example source code can be found here [GitHub](https://github.com/marzelwidmer/kboot-resilience4j)



> **_References:_**  
>[Resilience4j docs](https://resilience4j.readme.io/docs)


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

