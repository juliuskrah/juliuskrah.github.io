---
layout:     post
title:      Building an SMPP Application using Spring Boot and Camel
date:       2020-03-27 18:32:25 +0000
categories: blog
tags:       java spring-boot maven
section:    blog
author:     juliuskrah
repo:       smpp/tree/camel
---
> Short Message Peer-to-Peer (SMPP) is a protocol used by the telecommunications industry for exchanging
  messages between Short Message Service Centers (SMSC) and/or External Short Messaging Entities (ESME).

# Introduction

In this post I will build a simple app that will send SMS using the SMPP protocol, bootstraped with 
Spring Boot and [Camel][camel]{:target="_blank"}.

SMPP is a level-7 TCP/IP protocol, which allows fast delivery of SMS messages. The most conmmonly used
versions of SMPP are v3.3, the most widely supported standard, and v3.4, which adds transceiver support
(single connections that can send and receive messages).

# Prerequisites

- [Java Development Kit][JDK]{:target="_blank"}
- [Maven][]{:target="_blank"}
- SMPP account that supports v3.4

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__juliuskrah/
|  |  |  |  |  |__smpp/
|  |  |  |  |  |  |__Application.java
|  |  |  |  |  |  |__MessageReceiver.java
|  |  |  |  |  |  |__MessageRoute.java
|  |  |__resources/
|  |  |  |__application.yaml
|__pom.xml
```

## Project Setup

Create a simple Spring Boot Application by heading over to [start.spring.io](https://start.spring.io/#!type=maven-project&language=java&platformVersion=2.2.6.RELEASE&packaging=jar&jvmVersion=11&groupId=com.juliuskrah&artifactId=smpp&name=smpp&description=Demo%20project%20for%20Camel%20SMPP&packageName=com.juliuskrah.smpp){:target="_blank"}.
Add the following to your `pom.xml` (assuming you picked maven) after creating your Spring-Boot
application:

{% highlight xml %}
<!-- Sections omitted -->
<properties>
  <java.version>11</java.version>
  <camel-spring.version>3.1.0</camel-spring.version>
</properties>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.apache.camel.springboot</groupId>
      <artifactId>camel-spring-boot-dependencies</artifactId>
      <version>${camel-spring.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>org.apache.camel.springboot</groupId>
    <artifactId>camel-spring-boot-starter</artifactId>
  </dependency>
  <dependency>
    <groupId>org.apache.camel.springboot</groupId>
    <artifactId>camel-smpp-starter</artifactId>
  </dependency>
  <dependency>
    <groupId>org.apache.camel.springboot</groupId>
    <artifactId>camel-bean-starter</artifactId>
  </dependency>
</dependencies>
{% endhighlight %}

# Configuring Came SMPP

SMPP using camel is very covenient, as it provides some auto-configuration for Spring. We need the
following configuration to get started:

file: {% include file-path.html file_path='src/main/resources/application.yaml' %}

{% highlight yaml %}
camel:
  component:
    enabled: true
    smpp:
      configuration:
        host: :smpp.host:
        password: :password:
        system-id: :user:
        source-addr-ton: 0x05
        source-addr-npi: 0x01
        dest-addr-npi: 0x01
        dest-addr-ton: 0x01
        registered-delivery: 1
    bean:
      scope: singleton
  springboot:
    name: smpp-app
    main-run-controller: false
{% endhighlight %}

> NOTE: Be sure to replace the values for `host`, `system-id` and `password`. You can get these values 
  from your SMPP service provider.

That's all we need. Now let's add code to send an SMS

## Send an SMS

We will add a simple method that sends an SMS on application bootstrap:

{% highlight java %}
private Exchange sendTextMessage(ProducerTemplate template, String sourceAddress,
      String destinationAddress, String message) {
    var exchange = ExchangeBuilder.anExchange(context)
        .withHeader("CamelSmppDestAddr", List.of(destinationAddress))
        .withHeader("CamelSmppSourceAddr", sourceAddress)
        .withPattern(ExchangePattern.InOnly)
        .withBody(message).build();
    {% raw  %}
    // exceptions are not thrown from this method
    // exceptions are stored in Exchange#setException()
    return template.send("smpp://{{camel.component.smpp.configuration.host}}", exchange);
    {% endraw %}
}
{% endhighlight %}

With this method created, we can call it on application startup with a `CommandlineRunner`

{% highlight java %}
@Bean
CommandLineRunner init(ProducerTemplate template) {
    return args -> {
        var exchange = sendTextMessage(template, "5432", "<replace with phone number>", "Hello World!");
        if (exchange.getException() == null)
            log.info("Message Id - {}", exchange.getMessage().getHeader("CamelSmppId"));
        else
            log.error("Could not send message", exchange.getException());
    };
}
{% endhighlight %}

Putting it all together:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/smpp/Application.java' %}

{% highlight java %}
@SpringBootApplication
public class Application {
    private static final Logger log = LoggerFactory.getLogger(Application.class);
    @Autowired private CamelContext context;

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    private Exchange sendTextMessage(ProducerTemplate template, String sourceAddress,
        String destinationAddress, String message) {
        var exchange = ExchangeBuilder.anExchange(context)
            .withHeader("CamelSmppDestAddr", List.of(destinationAddress))
            .withHeader("CamelSmppSourceAddr", sourceAddress)
            .withPattern(ExchangePattern.InOnly)
            .withBody(message).build();
        {% raw  %}
        // exceptions are not thrown from this method
        // exceptions are stored in Exchange#setException()
        return template.send("smpp://{{camel.component.smpp.configuration.host}}", exchange);
        {% endraw %}
    }

    @Bean
    CommandLineRunner init(ProducerTemplate template) {
        return args -> {
            var exchange = sendTextMessage(template, "5432", "<replace with phone number>", "Hello World!");
            if (exchange.getException() == null)
                log.info("Message Id - {}", exchange.getMessage().getHeader("CamelSmppId"));
            else
                log.error("Could not send message", exchange.getException());
        };
    }

}
{% endhighlight %}

## Receive delivery receipts

After sending an SMS you may want to receive delivery receipts. To do this, we will create a bean
to process the delivery receipts.

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/smpp/MessageReceiver.java' %}

{% highlight java %}
@Component
public class MessageReceiver {
    private final static Logger log = LoggerFactory.getLogger(MessageReceiver.class);

    public void receive(Exchange exchange) {
        if (exchange.getException() == null) {
            var message = exchange.getIn();
            log.info("Received id {}", message.getHeader("CamelSmppId"));
            log.info("Text :- {}", message.getBody());
            log.info("Total delivered {}", message.getHeader("CamelSmppDelivered"));
            log.info("Message status {}", message.getHeader("CamelSmppStatus"));
            log.info("Submitted date {}", message.getHeader("CamelSmppSubmitDate", Date.class));
            log.info("Done date {}", message.getHeader("CamelSmppDoneDate", Date.class));
        } else
            log.error("Error receiving message", exchange.getException());
    }
}
{% endhighlight %}

Next we will create a router to route messages from the SMSC to our bean:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/smpp/MessageRoute.java' %}

{% highlight java %}
@Component
public class MessageRoute extends RouteBuilder {

    @Override
    public void configure() throws Exception {
        {% raw  %}
        from("smpp://{{camel.component.smpp.configuration.host}}")
                .to("bean:messageReceiver?method=receive");
        {% endraw  %}
    }   
}
{% endhighlight %}

# Conclusion

In this post we looked briefly at the SMPP protocol and its usage within the JVM. We also covered how to
use Spring Boot and Camel to make developing an SMPP client easier.

As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep
doing cool things :+1:.

[Camel]:                    https://camel.apache.org/components/latest/smpp-component.html
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org