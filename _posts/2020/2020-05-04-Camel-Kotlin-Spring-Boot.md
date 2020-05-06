---
layout: post
title: Camel with Kotlin and Spring Boot
description: Camel with Kotlin and Spring Boot
date: 2020-05-04 12:05:55 +0300
last_modified: 2020-05-04 12:05:55 +0300
image: k8s.png
tags: [Spring Boot, Kotlin]
--- 
# Apache Camel with Kotlin and Spring Boot
Apache Camel is an open source integration framework that empowers you to quickly and easily integrate various systems consuming or producing data.

### Precondition on OSX 
We will also use command line `ftp` commands for this you need the `ftp` command line tool this can be installed with :
```bash
brew install inetutils
```

## Create Project
Run the following commands :
```bash
export KBOOT_NAME=kboot-camel
export KBOOT_APPL_NAME=KbootCamel

http https://start.spring.io/starter.tgz \
    dependencies==camel,actuator,webflux \
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
### Change Maven Dependencies
When working with `Camel` best practice is to change the generated Maven `pom.xml` from the `start.spring.io`
Like in the official description of `Apache Camel` [camel-spring-boot](https://camel.apache.org/camel-spring-boot/latest/) 
So lets refactor our `pom.xml` 

We take the version from the `camel-spring-boot-starter` move it up to the `properties` section with the property name `camel.version`.
Add the `Camel` `BOM` `dependencieManagement` section to it.

#### Properties Section
```xml
<properties>`
		<camel.version>3.2.0</camel.version>
        ...
</properties>
```

#### Dependency Section
```xml
<!-- Camel -->
<dependency>
    <groupId>org.apache.camel.springboot</groupId>
    <artifactId>camel-spring-boot-starter</artifactId>
</dependency>
``` 
 

#### Dependency Management Section
```xml
<dependencyManagement>
    <dependencies>
        <!-- Camel BOM -->
        <dependency>
            <groupId>org.apache.camel.springboot</groupId>
            <artifactId>camel-spring-boot-dependencies</artifactId>
            <version>${camel.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## Start Application 
Let's check if `Camel` is loading in out Spring Boot application.

Run the following command :
```bash
mvn spring-boot:run
```

Verify the console output you should see something like `AbstractCamelContext - Apache Camel 3.2.0 (CamelContext: camel-1) is starting`

```bash
   _    ____              _
   | | __ __ )  ___   ___ | |_
   | |/ /  _ \ / _ \ / _ \| __|
   |   <| |_) | (_) | (_) | |_
   |_|\_\____/ \___/ \___/ \__|

:: kboot-camel: :: Running Spring Boot: (v2.3.0.RC1) :: Active Profiles: default ::

2020-05-05 Tue 07:50:57.489 KbootCamelKt                             - Starting KbootCamelKt on MacBookPro with PID 20731 (/Users/morpheus/dev/kboot-camel/target/classes started by morpheus in /Users/morpheus/dev/kboot-camel)
2020-05-05 Tue 07:50:57.493 KbootCamelKt                             - No active profile set, falling back to default profiles: default
2020-05-05 Tue 07:50:58.261 trationDelegate$BeanPostProcessorChecker - Bean 'org.apache.camel.spring.boot.CamelAutoConfiguration' of type [org.apache.camel.spring.boot.CamelAutoConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2020-05-05 Tue 07:50:58.548 LRUCacheFactory                          - Detected and using LURCacheFactory: camel-caffeine-lrucache
2020-05-05 Tue 07:50:58.739 EndpointLinksResolver                    - Exposing 2 endpoint(s) beneath base path '/actuator'
2020-05-05 Tue 07:50:58.971 SpringBootRoutesCollector                - Loading additional Camel XML routes from: classpath:camel/*.xml
2020-05-05 Tue 07:50:58.971 SpringBootRoutesCollector                - Loading additional Camel XML rests from: classpath:camel-rest/*.xml
2020-05-05 Tue 07:50:58.985 DefaultManagementStrategy                - JMX is enabled
2020-05-05 Tue 07:50:58.987 AbstractCamelContext                     - Apache Camel 3.2.0 (CamelContext: camel-1) is starting
2020-05-05 Tue 07:50:58.989 AbstractCamelContext                     - StreamCaching is not in use. If using streams then its recommended to enable stream caching. See more details at http://camel.apache.org/stream-caching.html
2020-05-05 Tue 07:50:58.989 AbstractCamelContext                     - Total 0 routes, of which 0 are started
2020-05-05 Tue 07:50:58.989 AbstractCamelContext                     - Apache Camel 3.2.0 (CamelContext: camel-1) started in 0.002 seconds
2020-05-05 Tue 07:50:59.139 NettyWebServer                           - Netty started on port(s): 8080
2020-05-05 Tue 07:50:59.142 KbootCamelKt                             - Started KbootCamelKt in 1.965 seconds (JVM running for 2.195)
```


## File Builder Route
```kotlin
@Component
class FileRouteBuilder : RouteBuilder() {

    private val workDir =System.getenv("PWD")
    private val input = "$workDir/orders/in?include=order.xml"
    private val output = "$workDir/orders/out?fileExist=Fail"

    @Throws(Exception::class)
    override fun configure() {
        from("file:$input")
            .process(HeaderProcessor())
            .to("file:$output")
            .log("Camel body: \${body.class} \${body}")

    }
}

```

To test the `FileBuilderRoute` you can copy the test `XML` files from my sample application on [GitHub](https://github.com/marzelwidmer/kboot-camel)
```bash
.
├── files
│   ├── order-2.xml
│   └── order.xml

```

in the folder `orders/in` after application start. The proceeded files will be then in the hidden folder `.camel`
```bash
orders
├── in
│   └── .camel
│       ├── order-2.xml
│       └── order.xml
└── out
    ├── 2019-01-28-order-2.xml
    └── order.xml
```

### Processor
Creat a `HeaderProcessor` that implement the function `process` from the interface `org.apache.camel.Processor`.
there we parse the date with `XPath`. 
```kotlin
class HeaderProcessor : Processor {

    private final val XPATH_DATE = "/order/orderDate/text()"

    override fun process(exchange: Exchange?) {
        val oderXml = exchange?.`in`?.body
        val orderDateTime = XPathBuilder.xpath(XPATH_DATE).evaluate(exchange?.context, oderXml)
        val formattedOrderDate = getFormattedData(orderDateTime = orderDateTime)
        exchange?.`in`?.setHeader("orderDate", formattedOrderDate)
        exchange?.`in`?.setHeader("uuid", UUID.randomUUID().toString().take(4))
    }

    // TODO: 05.05.20 DirtyHarry Implementation
    private fun getFormattedData(orderDateTime: String) = DateTimeFormatter.ofPattern("yyyy-MM-dd")
        .format(LocalDateTime.ofInstant(Instant.parse(orderDateTime), ZoneOffset.UTC))
}
```

Add `HeaderProcessor` class to our `FileRouteBuilder` with `.process(HeaderProcessor())`
We also change the filename in the `to` section with `.to("file:$workDir/orders/out?fileName=\${header.orderDate}-\${header.CamelFileName}")`

```kotlin
@Component
class FileRouteBuilder : RouteBuilder() {

    private val workDir =System.getenv("PWD")
    private val input = "$workDir/orders/in?include=order.xml"
    private val output = "$workDir/orders/out?fileExist=Fail"

    @Throws(Exception::class)
    override fun configure() {
        from("file:$input")
            .to("file:$output")
            .log("Camel body: \${body.class} \${body}")

        from("file:$workDir/orders/in?include=order-.*xml")
            .process(HeaderProcessor())
            .to("file:$workDir/orders/out?fileName=\${header.orderDate}-\${header.uuid}-\${header.CamelFileName}")
            .log("Camel body: \${body.class} \${body}")
    }
}

```



## FTP Route Builder
```kotlin
@Component
class FtpRouteBuilder : RouteBuilder() {


    private val ftpEndpoint ="ftp.walkerit.ch"
    private val username ="public"
    private val password ="Public8852"

    private val workDir =System.getenv("PWD")
    private val output = "$workDir/orders/out?fileExist=Fail"


    @Throws(Exception::class)
    override fun configure() {
         from("ftp://$ftpEndpoint?username=$username&password=$password&delete=true&include=order.*xml")
             .log("New File \${header.CamleFileName} picked up from \${header.CamleFileHost}")
             .process(ExchangePrinter())
             .to("file://$output")
    }
}
```

### Processor
```kotlin
class ExchangePrinter : Processor {

    private val log = LoggerFactory.getLogger(javaClass)

    @Throws(Exception::class)
    override fun process(exchange: Exchange?) {
        val body = exchange?.`in`?.body
        log.info("Body: $body")
    }
}
```

## FTP Server
Upload test `xml` file to public `FTP` server [https://www.walkerit.ch/public-ftp](https://www.walkerit.ch/public-ftp)
with the script `ftp-upload.sh`.
 
Run the following command :
```bash
./ftp-upload.sh
```

## Start Application
When you start the Application `mvn spring-boot:run` you will some information now in the console like :
```bash
2020-05-06 Wed 07:32:39.101 KbootCamelKt    - Started KbootCamelKt in 2.055 seconds (JVM running for 2.265)
2020-05-06 Wed 07:32:40.099 route3          - New File  picked up from
2020-05-06 Wed 07:32:40.099 ExchangePrinter - Body: RemoteFile[order-ftp.xml]
```

and the files will be downloaded in the `out` folder.
```bash
 tree
.
├── in
└── out
    └── order-ftp.xml
```




> **_References:_**  
>[GitHub Sample Project](https://github.com/marzelwidmer/kboot-camel)
>[Spring Tips Apache Camel](https://spring.io/blog/2018/05/23/spring-tips-apache-camel)
>[Camel Apache Spring Boot](https://camel.apache.org/camel-spring-boot/latest/)
>[Public FTP](https://www.walkerit.ch/public-ftp)
 

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

