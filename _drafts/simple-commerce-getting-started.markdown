---
layout:     series
title:      'Simple Commerce, Part 1: Getting Started'
categories: tutorial
---


Using the Spring Starter download a project

```bash
curl -G https://start.spring.io/starter.zip -d language=java -d platformVersion=3.3.1 -d packaging=jar -d jvmVersion=21 -d groupId=com.simplecommerce -d artifactId=simple-commerce -d name=simple-commerce -d description=Simple%20Commerce%20GraphQL%20API -d packageName=com.simplecommerce -d dependencies=web -d type=gradle-project-kotlin -o simple-commerce.zip
```

```bash
unzip simple-commerce.zip
```

To unpack

```bash
curl -G https://start.spring.io/starter.tgz -d language=java -d platformVersion=3.3.1 -d packaging=jar -d jvmVersion=21 -d groupId=com.simplecommerce -d artifactId=simple-commerce -d name=simple-commerce -d description=Simple%20Commerce%20GraphQL%20API -d packageName=com.simplecommerce -d dependencies=web -d type=gradle-project-kotlin -d baseDir=simple-commerce | tar -xzvf -
```

Add the hello world route

```java
@Bean
RouterFunction<?> home() {
    return RouterFunctions.route()
            .GET("/", request -> ServerResponse.ok().body("Hello World!"))
            .build();
}

@Bean
RouterFunction<?> index() {
    return RouterFunctions.route()
            .GET("/index", request -> ServerResponse.permanentRedirect(URI.create("/"))
            .build())
            .build();
}
```