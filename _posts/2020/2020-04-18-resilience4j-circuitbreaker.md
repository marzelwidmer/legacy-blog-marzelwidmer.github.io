--- 
layout: post
title: Reactive Spring Boot with Resilience4j CircuitBreaker
description: Resilience4j - CircuitBreaker with Kotlin Reactive Spring Boot Application
date:   2020-04-18 12:05:55 +0300
last_modified:   2020-04-18 12:05:55 +0300
image: IMG_0355.jpg
tags: [Spring Boot, Kotlin]
---
# Reactive Spring Boot with Resilience4j CircuitBreaker

## Create Project
Let's create a Sample Application with Kotlin and Reactive Spring Boot with the [spring inializr](https://start.spring.io/) Rest Endpoint. We will take the latest and greates Spring Boot version `2.3.0.M4` and language `kotlin` with the following dependencies:
* actuator
* webflux
* cloud-resilience4j

```
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
http https://raw.githubusercontent.com/marzelwidmer/marzelwidmer.github.io/master/banner.txt \
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

## Domain Model
Let's start with the `Movie` domain class with the following properties.
* name
* year
* description

```kotlin
data class Movie(val id: String? = UUID.randomUUID().toString(), val name: String, val year: Year, val description: String)
``` 

## Service 
Now we create the service class `MovieService` who hold some hard coded movies in a list.
The amazing functions:

* Get all Movies
* Get a random list of Movies
* Get a Movie by his name
* Get a Movie by his ID

```kotlin
@Service
class MovieService {

    private val movies = listOf(
        Movie(name = "Matrix",
            year = Year.of(1999),
            description = "A computer hacker learns from mysterious rebels about the true nature of his reality and his role in the war against its controllers."),
        Movie(name = "The Godfather",
            year = Year.of(1972),
            description = "The aging patriarch of an organized crime dynasty transfers control of his clandestine empire to his reluctant son."),
        Movie(name = "Casablanca",
            year = Year.of(1942),
            description = "A cynical American expatriate struggles to decide whether or not he should help his former lover and her fugitive husband escape French Morocco."),
        Movie(name = "Rocky",
            year = Year.of(1976),
            description = "A small-time boxer gets a supremely rare chance to fight a heavy-weight champion in a bout in which he strives to go the distance for his self-respect.")
    )

    fun randomMovie() = Mono.just(movies[kotlin.random.Random.nextInt(movies.size)])
    fun movies() = Flux.just(movies)
    fun movieById(id: String) = Mono.just(movies.first { it.id == id })
    fun movieByName(name: String?): Mono<Movie> {
            name?.map {
                movies.firstOrNull() { it.name.toLowerCase() == name.toLowerCase() }?.let {
                    return Mono.just(it)
                }
            }.isNullOrEmpty().apply {
                return Mono.error(IllegalArgumentException("Movie was not found."))
            }
    }

}
```
## Rest API  
I think now is time to create a REST API `/movies/random` with the [Reactive router Kotlin DSL](https://docs.spring.io/spring-framework/docs/current/kdoc-api/spring-framework/org.springframework.web.reactive.function.server/-router-function-dsl/index.html). 
A easy way to create a `WebFlux.fn` [RouterFunction](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/reactive/function/server/RouterFunctions.html)
Because we have more then one endpoint under `/movies` we use the `"/movies".nest` router function.


We use also our service `MovieService` so we need a [Bean Reference](https://docs.spring.io/spring/docs/current/kdoc-api/spring-framework/org.springframework.context.support/-bean-definition-dsl/-bean-supplier-context/ref.html) to it.

`val service = ref<MovieService>()`

> 💡: Take care of the order in the Route definiton "/" have to be at latest position.

```kotlin
fun main(args: Array<String>) {
    runApplication<Resilience4jApplication>(*args) {
        addInitializers(
            beans {
                bean {
                    router {
                         "movies".nest {
                             val service = ref<MovieService>()
                             //http :8080/movies/random
                             GET("/random") {
                                 ok().body(service.randomMovie())
                             }
                             //http :8080/movies/ name=="Rocky" -vv
                             queryParam("name", { true }) {
                                 ok().body(service.movieByName(name = it.queryParam("name").get()))
                             }
                             //http :8080/movies/c7f399bc-ff4c-4a2f-bddf-d92d53a96df2
                             GET("/{id}") {
                                 ok().body(service.movieById(id = it.pathVariable("id")))
                             }
                             //http :8080/movies/random
                             GET("/") {
                                 ok().body(service.movies())
                             }
                         }
                     }
                }
            }
        )
    }
}
```

## Test Rest API
No is time to make some calls from the terminal with the `HTTPie` or in a Browser.
First start the application e.g. with `mvn spring-boot:run`.

Then lets call our amazing endpoints with the `HTTPie` or Browser.

### Get all Movies

[http://localhost:8080/movies](http://localhost:8080/movies)

```bash
http :8080/movies/
HTTP/1.1 200 OK
Content-Type: application/json
transfer-encoding: chunked
[
    [
        {
            "description": "A computer hacker learns from mysterious rebels about the true nature of his reality and his role in the war against its controllers.",
            "id": "c7f399bc-ff4c-4a2f-bddf-d92d53a96df2",
            "name": "Matrix",
            "year": "1999"
        },
        {
            "description": "The aging patriarch of an organized crime dynasty transfers control of his clandestine empire to his reluctant son.",
            "id": "0c7a74fc-b735-41a3-a383-72b9dad7d608",
            "name": "The Godfather",
            "year": "1972"
        },
        {
            "description": "A cynical American expatriate struggles to decide whether or not he should help his former lover and her fugitive husband escape French Morocco.",
            "id": "15a9e15d-4a5e-4b8a-a7c5-d34b0d5d0879",
            "name": "Casablanca",
            "year": "1942"
        },
        {
            "description": "A small-time boxer gets a supremely rare chance to fight a heavy-weight champion in a bout in which he strives to go the distance for his self-respect.",
            "id": "f6fb4a62-6d84-434e-8e2c-70e4c1e7ab2c",
            "name": "Rocky",
            "year": "1976"
        }
    ]
]
```

### Get a random list of Movies
   
[http://localhost:8080/movies/random](http://localhost:8080/movies/random)

```bash
 http :8080/movies/random
HTTP/1.1 200 OK
Content-Length: 240
Content-Type: application/json

{
    "description": "A cynical American expatriate struggles to decide whether or not he should help his former lover and her fugitive husband escape French Morocco.",
    "id": "5dd310b8-8d51-4a1e-a20c-790fec00029f",
    "name": "Casablanca",
    "year": "1942"
}
```
 
### Get a Movie by his name
   
[http://localhost:8080/movies/?name=Rocky](http://localhost:8080/movies/?name=Rocky)

```bash
http :8080/movies name=="Rocky" -v
GET /movies?name=Rocky HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Connection: keep-alive
Host: localhost:8080
User-Agent: HTTPie/2.0.0

HTTP/1.1 200 OK
Content-Length: 242
Content-Type: application/json

{
    "description": "A small-time boxer gets a supremely rare chance to fight a heavy-weight champion in a bout in which he strives to go the distance for his self-respect.",
    "id": "5cae75c6-d18a-495b-a80f-60bdd0763eb1",
    "name": "Rocky",
    "year": "1976"
}
```

### Get a Movie by his ID
   
[http://localhost:8080/movies/5dd310b8-8d51-4a1e-a20c-790fec00029f](http://localhost:8080/movies/5dd310b8-8d51-4a1e-a20c-790fec00029f)

```bash
http :8080/movies/5dd310b8-8d51-4a1e-a20c-790fec00029f
HTTP/1.1 200 OK
Content-Length: 240
Content-Type: application/json

{
    "description": "A cynical American expatriate struggles to decide whether or not he should help his former lover and her fugitive husband escape French Morocco.",
    "id": "5dd310b8-8d51-4a1e-a20c-790fec00029f",
    "name": "Casablanca",
    "year": "1942"
}
```

### Search for a not existing Movie
Now let's also search for `http://localhost:8080/movies/?name=Creed` is also a great movie but this one is not yet in our `MovieService` included
we will get the following exception.
```bash
http :8080/movies name=="Creed" -v
GET /movies?name=Creed HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Connection: keep-alive
Host: localhost:8080
User-Agent: HTTPie/2.0.0

HTTP/1.1 500 Internal Server Error
Content-Length: 165
Content-Type: application/json

{
    "error": "Internal Server Error",
    "message": "Movie was not found.",
    "path": "/movies",
    "requestId": "61569a89-3",
    "status": 500,
    "timestamp": "2020-04-19T21:13:10.403+00:00"
}

```

## Configure CircuitBreaker

😎 Cool stuff 😎 let's implement the `CircuitBreaker` with `Resilinece4j`.
Let's configure the `ReactiveCircuitBreaker` Bean from `ReactiveResilience4JCircuitBreakerFactory` with a name `movie-service`.

```kotlin
bean {
    ReactiveResilience4JCircuitBreakerFactory()
        .create("movie-service")
}
```







😎 Cool stuff 😎 let's implement the `CircuitBreaker` with `Resilinece4j`. 

For this we create a `ReactiveCircuitBreaker` Bean from `ReactiveResilience4JCircuitBreakerFactory` with a name `readySetGo`.

```kotlin
bean {
    ReactiveResilience4JCircuitBreakerFactory()
        .create("readySetGo")
}
```
  


The example source code can be found here [GitHub](https://github.com/marzelwidmer/kboot-resilience4j)



> 💡 **Logger Configuration**: 
    logging.pattern.console: "%clr(%d{yyyy-MM-dd E HH:mm:ss.SSS}){blue} %clr(%-40.40logger{0}){magenta} - %clr(%m){green}%n" 
    

> **_References:_**  
>[Resilience4j docs](https://resilience4j.readme.io/docs)
 

 

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

