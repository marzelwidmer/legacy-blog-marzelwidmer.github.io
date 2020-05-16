---
layout: post
title: Flux meets SOAP - Non-Blocking I/O - Blocking I/O
description: Leading edge meets legacy
date: 2020-05-05 10:05:55 +0300
last_modified: 2020-05-10 13:05:55 +0300
image:  2020/flux-meets-soap/FluxMeetsSoap.png
tags: [Spring Boot, Kotlin, WebFlux]
--- 

This will demonstrate how we can deal with a `Blocking API` in a `Reactive World`.

This Sample provides a 
* `soap-server` who demonstrate the blocking downstream `API`.
* `flux-client` with `REST API` 
  * `lockdown` that will call the blocking `SOAP` endpoint and. [Blockhound](https://github.com/reactor/BlockHound) will throw an exception.
  * `easing` have an implemented from [Avoiding Reactor Meltdown](https://www.youtube.com/watch?v=xCu73WVg8Ps&t=7s) show case how to manage `Blocking API`.

With this approach to manage `Blocking API` in the same service ant not in a separate service we have all the nice features like `retry` `filter` `map` and so on
in our `Servive A` from the `Reactive Streams API`. 

We also not have to manage a other service who will handle it for us. With this we have less network hops, Organisations-Issues, Deployment etc. who are sometimes increase complexity and so on.


* [Rest API](#restAPI)
* [BlockHound Plugin](#blockHound)
* [SOAP Server](#soapServer)
* [SOAP with HTTPie Server](#httpieSoapCall)
* [Implementation](#implementation)


![hetzner-preis](/img/2020/flux-meets-soap/FluxMeetsSoap.png)


# Flux Client  <a name="restAPI"></a>
## API Lockdown Switzerland
```bash
http :8080/api/lockdown/Switzerland
```

## API easing Switzerland
```bash
http :8080/api/easing/Switzerland
```


## Blockhound  <a name="blockHound"></a>
[BlockHound](https://github.com/reactor/BlockHound) is a Java agent to detect blocking calls from non-blocking threads. 
Add the following or latest dependency from `blockhound`.
```xml
<!-- Blockhound	-->
<dependency>
    <groupId>io.projectreactor.tools</groupId>
    <artifactId>blockhound</artifactId>
    <version>1.0.3.RELEASE</version>
</dependency>
```

Install the agent `BlockHound.install()`
```kotlin
fun main(args: Array<String>) {
	BlockHound.install()
	runApplication<KbootFluxWS>(*args)
}
```
 
# SOAP Server <a name="soapServer"></a>
The Server have an implementation with a demonstration how we can write own `Kotlin DSL`.

## DSL
```kotlin
 country {
    name = "Switzerland"
    capital = "Bern"
    population = 8_603_900
    currency = "CHF"
}
```


## WSDL 
`http://localhost:8888/ws/countries.wsdl`

## End-Point
`http://localhost:8888/ws`

## Request 
```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
				  xmlns:gs="http://keepcalm.ch/web-services">
   <soapenv:Header/>
   <soapenv:Body>
      <gs:getCountryRequest>
         <gs:name>Switzerland</gs:name>
      </gs:getCountryRequest>
   </soapenv:Body>
</soapenv:Envelope>
```

## Call Service with `HTTPie` <a name="httpieSoapCall"></a>

```bash
printf '<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
                                  xmlns:gs="http://keepcalm.ch/web-services">
   <soapenv:Header/>
   <soapenv:Body>
      <gs:getCountryRequest>
         <gs:name>Switzerland</gs:name>
      </gs:getCountryRequest>
   </soapenv:Body>
</soapenv:Envelope>'| http  --follow --timeout 3600 POST http://localhost:8888/ws \
 Content-Type:'text/xml'
```




# Implementation <a name="implementation"></a>  
## Router Table 
```kotlin
bean {
    router {
        "api".nest {
            GET("/lockdown/{name}") {
                val countryService = ref<CountryService>()
                ok().body(BodyInserters.fromValue(
                    countryService.getCountryByName(it.pathVariable("name")))
                )
            }
            GET("/easing/{name}") {
                val countryReactiveService = ref<CountryReactiveService>()
                ok().body(
                    BodyInserters.fromPublisher(countryReactiveService.getCountryByName(it.pathVariable("name")), GetCountryResponse::class.java)
                )
            }
        }
    }
}
```

## Reactive Service
```kotlin
@Service
class CountryReactiveService  (private val soapClient: SoapClient) {

    fun getCountryByName(name: String): Mono<GetCountryResponse> {
        return soapClient.getCountryReactive(name)
    }
}
```
## Reactive SOAP Client
`.subscribeOn(Schedulers.boundedElastic())`
```kotlin
fun getCountryReactive(country: String): Mono<GetCountryResponse> {
    val request = GetCountryRequest()
    request.name = country
    log.info("Requesting location for $country")
    return Mono.fromCallable {
        webServiceTemplate
            .marshalSendAndReceive("http://localhost:8888/ws/countries", request,
                SoapActionCallback(
                    "http://keepcalm.ch/web-services/GetCountryRequest")) as GetCountryResponse
    }
        // properly schedule above blocking call on
        // scheduler meant for blocking tasks
        .subscribeOn(Schedulers.boundedElastic())
}
```

> **_References:_**  
* [Kotlin DSL in under an hour](https://www.youtube.com/watch?v=zYNbsVv9oN0)
* [Do Super Language with Kotlin](https://www.youtube.com/watch?v=hYXAFO3q3qU)
* [Blockhound](https://github.com/reactor/BlockHound)
* [Avoiding Reactor Meltdown](https://youtu.be/xCu73WVg8Ps?t=1)
* [Avoiding Reactor Meltdown GitHub](https://github.com/philsttr/avoiding-reactor-meltdown)
 



[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

