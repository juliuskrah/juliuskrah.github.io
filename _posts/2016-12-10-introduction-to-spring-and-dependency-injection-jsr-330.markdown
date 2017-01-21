---
layout:     post
title:      'Introduction to Spring and Dependency Injection (JSR 330)'
date:       2016-12-10 00:08:31 +0000
categories: blog
tags:       java spring maven
section:    blog
author:     Julius Krah
repo:       gs-spring
---
> The [Spring Framework][Spring]{:target="_blank"} provides a comprehensive programming and configuration model for modern Java-based 
  enterprise applications - on any kind of deployment platform. A key element of Spring is infrastructural support at the application 
  level: Spring focuses on the "plumbing" of enterprise applications so that teams can focus on application-level business logic, 
  without unnecessary ties to specific deployment environments.

# Introduction
Spring Framework is an implementation of the [Inversion of Control][IoC]{:target="_blank"} (IoC) principle. IoC is also known as 
`Dependency Injection` (DI). It is a process whereby objects define their dependencies, that is, the other objects they work with, 
only through constructor arguments, arguments to a factory method, or properties that are set on the object instance after it is 
constructed or returned from a factory method. 

> The objects that form the backbone of your application and that are managed by the Spring IoC container are called `beans`. 
  A `bean` is an object that is instantiated, assembled, and otherwise managed by a Spring IoC container. These beans are created 
  with the configuration metadata that you supply to the container, for example, in the form of `@Bean` definitions.

The container then injects those dependencies when it creates the `bean`. This process is fundamentally the inverse, hence the name 
`Inversion of Control` (IoC), of the `bean` itself controlling the instantiation or location of its dependencies by using direct 
construction of classes, or a mechanism such as the _Service Locator_ pattern.

## Pre-requisites
- [Java Development Kit][JDK]{:target="_blank"}  
- [Maven][]{:target="_blank"}

## Project Structure
To demonstrate Dependency Injection, we will create a project that showcases the concept. This project will have the following 
directory structure:

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__juliuskrah/
|  |  |  |  |  |__gs/
|  |  |  |  |  |  |__Application.java
|  |  |  |  |  |  |__MessagePrinter.java
|  |  |  |  |  |  |__MessageService.java
|__pom.xml
```

You can also find the complete sample {% include source.html %}.

# Dependency Injection
Every java based application has a few objects that work together to present what the end-user sees as a working application. 
When writing a complex Java application, application classes should be as independent as possible of other Java classes to 
increase the possibility to reuse these classes and to test them independently of other classes while doing unit testing. 
Dependency Injection helps in gluing these classes together and at the same time keeping them independent.

Consider you have an application which has a printing component and you want to provide messages to be printed. 
Your standard code would look something like this:

{% highlight java %}
public class MessagePrinter {
  private MessageService messageService;
	
  public MessagePrinter() {
    messageService = new MessageServiceImpl();
  }

  public void printMessage() {
    System.out.println(this.messageService.getMessage());
  }
}
{% endhighlight %}

{% highlight java %}
@FunctionalInterface
public interface MessageService {
  String getMessage();
}
{% endhighlight %}

What we've done here is create a dependency between the `MessagePrinter` and the `MessageService`. In an inversion of control 
scenario we would instead do something like this:

{% highlight java %}
public class MessagePrinter {
  private MessageService messageService;
	
  public MessagePrinter(MessageService messageService) {
    this.messageService = messageService;
  }

  public void printMessage() {
    System.out.println(this.messageService.getMessage());
  }
}
{% endhighlight %}

Here `MessagePrinter` should not worry about `MessageService` implementation. The `MessageService` will be implemented 
independently and will be provided to `MessagePrinter` at the time of `MessagePrinter` instantiation and this entire procedure 
is controlled by the Spring Framework.

At this point we will declare our dependencies on the Spring Framework. This a [Maven][]{:target="_blank"} based project and we 
will use a `pom.xml` to manage our dependencies:

{% highlight xml %}
...
<properties>
  <spring.version>4.3.4.RELEASE</spring.version>
</properties>

<dependencies>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>${spring.version}</version>
  </dependency>
  <dependency>
    <groupId>javax.inject</groupId>
    <artifactId>javax.inject</artifactId>
    <version>1</version>
  </dependency>
</dependencies>
...
{% endhighlight %}

In this post we will be using the Standard `JSR 330` (found in `javax.inject` package) annotations to demonstrate DI.

# Adding Spring and JSR 330 Annotations
Before we dive into the cool things, let us look at the various options for injection in Spring:

1. Constructor-based dependency injection
2. Setter-based dependency injection
3. Field-based dependency injection

We will demonstrate all three with code examples.

The above `MessagePrinter` class can be rewritten to add Spring support using constructor-based injection:

{% highlight java %}
import javax.inject.Inject;

import org.springframework.stereotype.Component;

@Component
public class MessagePrinter {
  private MessageService messageService;

  @Inject
  public MessagePrinter(MessageService messageService) {
    this.messageService = messageService;
  }

  public void printMessage() {
    System.out.println(this.messageService.getMessage());
  }
}
{% endhighlight %}

Notice the introduction of two new annotations; `@Component` at the class or type level and `@Inject` at the constructor level. 

> **NOTE**  
  Since [Spring 4.3][4.3]{:target="_blank"} it is no longer necessary to specify an injection point if the target bean only 
  defines one constructor. In our case, the `@Inject` annotation is not necessary as we have only one constructor.


The example above shows the basic concept of dependency injection, the `MessagePrinter` is decoupled from the `MessageService` 
implementation, with Spring Framework wiring everything together.

> `@Component` indicates that an annotated class is a "component". Such classes are considered as candidates for auto-detection 
  when using annotation-based configuration and classpath scanning.   
  `@Inject` identifies injectable constructors, methods, and fields. May apply to static as well as instance members. An 
  injectable member may have any access modifier (private, package-private, protected, public). Constructors are injected 
  first, followed by fields, and then methods. Fields and methods in superclasses are injected before those in subclasses. 
  Ordering of injection among fields and among methods in the same class is not specified.

The `MessagePrinter` class can be rewritten to add Spring support using setter-based injection:

{% highlight java %}
import javax.inject.Inject;

import org.springframework.stereotype.Component;

@Component
public class MessagePrinter {
  private MessageService messageService;

  public MessageService getMessageService() {
    return messageService;
  }

  @Inject
  public void setMessageService(MessageService messageService) {
    this.messageService = messageService;
  }

  public void printMessage() {
    System.out.println(this.messageService.getMessage());
  }
}
{% endhighlight %}

The `MessagePrinter` class can be rewritten to add Spring support using field-based injection:

{% highlight java %}
import javax.inject.Inject;

import org.springframework.stereotype.Component;

@Component
public class MessagePrinter {
  @Inject
  private MessageService messageService;

  public void printMessage() {
    System.out.println(this.messageService.getMessage());
  }
}
{% endhighlight %}


Now that we have most of our application configuration metadata out of the way, we can create our configuration class:

{% highlight java linenos %}
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan
public class Application {

  @Bean
  public MessageService messageService() {
    return () -> "Hello World!";
  }
	
  public static void main(String... args) {
    ApplicationContext applicationContext = new AnnotationConfigApplicationContext(Application.class);
    MessagePrinter messagePrinter =  applicationContext.getBean(MessagePrinter.class);

    messagePrinter.printMessage();
  }
}
{% endhighlight %}

From the above snippet on lines 7, 8 and 11 we see three new annotations `@Configuration`, `@ComponentScan`, and 
`@Bean` respectively.

> `@Configuration` indicates that a class declares one or more `@Bean` methods and may be processed by the Spring container 
  to generate bean definitions and service requests for those beans at runtime.  
  `@ComponentScan` configures component scanning directives for use with `@Configuration` classes.  
  `@Bean` indicates that a method produces a bean to be managed by the Spring container.

Before we go any further, a few more notes on `@ComponentScan` as it applies in this guide and Spring in general.

1.    Either `basePackageClasses` or `basePackages` (or its alias value) may be specified to define specific packages to scan. 
      If specific packages are not defined, scanning will occur from the package of the class that declares this annotation 
      (in our case `com.juliuskrah.gs`). What this means is, you can specify a package that Spring needs to scan for `Component`s.
2.    The `MessagePrinter` class is annotated with `@Component` which makes it eligible for discovery by the `@ComponentScan` as
      it is also located in the `com.juliuskrah.gs` package.
3.    `MessagePrinter` now scanned by the Spring container can now process `@Inject` annotations.

On line 17 of `Application.java` we have the `ApplicationContext`. 

> The `ApplicationContext` is the central interface within a Spring application for providing configuration information to the 
  application. It is read-only at run time, but can be reloaded if necessary and supported by the application. A number of 
  classes implement the `ApplicationContext` interface (e.g. `AnnotationConfigApplicationContext`), allowing for a variety 
  of configuration options and types of applications.

The ApplicationContext provides:

1.    Bean factory methods for accessing application components.
2.    The ability to load file resources in a generic fashion.
3.    The ability to publish events to registered listeners.
4.    The ability to resolve messages to support internationalization.
5.    Inheritance from a parent context.

On line 18, we call `getBean()` and pass `MessagePrinter` as argument. This is possible because, the `@ComponentScan` has detected
the `@Component` annotation on `MessagePrinter` and registered it as a `Bean`. The Spring container then processes the injected
constructor argument `MessageService` and looks for a `Bean` of type `MessageService`. This `Bean` is defined in the `Application`
class implemented to return `"Hello World!"` for the `getMessage()` method.

When this project is run, it will print `"Hello World!"` in the console.

# Conclusion
In this post we learnt the basics of the Spring Framework and Dependency Injection using code samples. We also talked about the 
types of injection in Spring. You can mix, Constructor-based, Setter-based and Field-based DI but it is a good rule of thumb to use 
constructor arguments for mandatory dependencies and setters or field for optional dependencies.
You can find the source to this guide {% include source.html %}. Until the next post, keep doing cool things :smile:.

[Spring]: http://projects.spring.io/spring-framework/
[JSR-330]: https://jcp.org/en/jsr/detail?id=330
[IoC]: https://docs.spring.io/spring/docs/current/spring-framework-reference/html/overview.html#background-ioc
[Maven]: http://maven.apache.org/
[4.3]: http://docs.spring.io/spring/docs/4.3.x/spring-framework-reference/htmlsingle/#new-in-4.3
[JDK]: http://www.oracle.com/technetwork/java/javase/downloads/index.html