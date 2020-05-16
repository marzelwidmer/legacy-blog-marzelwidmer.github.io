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

This Sample provides a `soap-server` who demonstrate the blocking downstream `API`.
The `flux-client` have two `REST API` who call the blocking `SOAP` endpoint the `lockdown` with no special implementation,
where the [Blockhound](https://github.com/reactor/BlockHound) will throw an exception. 

The `easing` `REST API` have implemented [Avoiding Reactor Meltdown](https://www.youtube.com/watch?v=xCu73WVg8Ps&t=7s) show case how to manage `Blocking API`.

With this approach to manage `Blocking API` in the same service in not in a separate service we have all the nice features like `retry` `filter` `map` and so on in our `Servive A` from the `Reactive Streams API`. 
We also not have to manage no other service and have less network hops who are sometimes increase complexity and so on.


![hetzner-preis](/img/2020/flux-meets-soap/FluxMeetsSoap.png)


# Flux Client
## API Lockdown Switzerland
```bash
http :8080/api/lockdown/Switzerland
```

## API easing Switzerland
```bash
http :8080/api/easing/Switzerland
```


## Blockhound
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

# SOAP Server
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

## EndPoint
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

## Call Service with `HTTPie`

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

 
> **_References:_**  
>[Kotlin DSL in under an hour](https://www.youtube.com/watch?v=zYNbsVv9oN0)
>[Do Super Language with Kotlin](https://www.youtube.com/watch?v=hYXAFO3q3qU)
>[Blockhound](https://github.com/reactor/BlockHound
>[Avoiding Reactor Meltdown](https://youtu.be/xCu73WVg8Ps?t=1)
>[Avoiding Reactor Meltdown GitHub](https://github.com/philsttr/avoiding-reactor-meltdown)
 


 

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

