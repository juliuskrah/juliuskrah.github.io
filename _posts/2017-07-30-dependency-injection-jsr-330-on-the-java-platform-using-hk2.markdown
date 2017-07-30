---
layout:     post
title:      Dependency Injection (JSR 330) on the Java platform using HK2
date:       2017-07-30 22:17:09 +0000
categories: blog
tags:       java maven javaee
section:    blog
author:     juliuskrah
repo:       gs-spring/tree/hk2
---
> [`HK2`][HK2]{:target="_blank"} is an implementation of [`JSR-330`][JSR-330]{:target="_blank"} in a JavaSE 
  environment.  
  `JSR-330` defines services and injection points that can be dynamically discovered at runtime and which allow for 
  `Inversion of Control` (IoC) and `dependency injection` (DI).

# Introduction
Most developers are familiar with using the [Spring Container for dependency injection]({% post_url 2016-12-10-introduction-to-spring-and-dependency-injection-jsr-330 %}),
in this post we are going to look at dependency injection using `HK2`.

## Prerequisites
To follow along with this post, your development environment must satisfy the following requirements:

- [Java Development Kit][JDK]{:target="_blank"}  
- [Maven][]{:target="_blank"}

## Project Structure
At the end of this post, you should have a project structure similar to the one illustrated below:

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
|  |  |  |  |  |  |__MessageServiceImpl.java
|__pom.xml
```

You can also find the complete sample {% include source.html %}.

## Project Dependencies
To get started, we need to gather our project dependencies. This is a maven based project, and we can easily declare 
our dependencies in a `pom.xml` file.

file: {% include file-path.html file_path='pom.xml' %}

{% highlight xml %}
...
<properties>
  <maven.compiler.source>1.8</maven.compiler.source>
  <maven.compiler.target>1.8</maven.compiler.target>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <hk2.version>2.5.0-b43</hk2.version>
</properties>

<dependencies>
  <dependency>
    <groupId>org.glassfish.hk2</groupId>
    <artifactId>hk2-locator</artifactId>
    <version>${hk2.version}</version>
  </dependency>
  <dependency>
    <groupId>org.glassfish.hk2</groupId>
    <artifactId>hk2-metadata-generator</artifactId>
    <version>${hk2.version}</version>
    <scope>provided</scope>
  </dependency>
</dependencies>
{% endhighlight %}

We have declared two dependencies, `hk2-locator` and `hk2-metadata-generator`. The first dependency contains the hk2
library and its transitive dependencies. We will talk about the second dependency later in this post.

# HK2 Services and Injection
To demonstrate dependency injection using HK2, we will create a concrete `Service` class `MessagePrinter` and inject
a `Contract` interface `MessageService`. The `MessagePrinter` will print messages delivered via the `MessageService`:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/gs/MessageService.java' %}

{% highlight java %}
@Contract
@FunctionalInterface
public interface MessageService {
  String getMessage();
}
{% endhighlight %}

The `MessageService` interface defines one contract `getMessage` which returns a `String`. The `MessagePrinter` will
inject an instance of `MessageService` and print its messages:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/gs/MessagePrinter.java' %}

{% highlight java linenos %}
@Service
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

On line 5 we use constructor injection to inject the `MessageService` instance.

Next we create a concrete service implementation of `MessageService`. It is this service implementation that
HK2 will inject into the `MessageService` contract.

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/gs/MessageServiceImpl.java' %}

{% highlight java %}
@Service
public class MessageServiceImpl implements MessageService {
    
    @Override
    public String getMessage() {
        return "Hello World!";
    }
}
{% endhighlight %}

Now that we have our services and contract set up, we wire it all together using HK2.  
In order for HK2 to automatically find services at runtime it can read files called `inhabitant files`. These are 
usually placed in your JAR file at location `META-INF/hk2-locator`. Normally the file is named `default`. (You can 
however use a different file name or location(s) by using more specific API). HK2 has a tool for automatically 
creating these files based on class files annotated with `@Service`. There is also a simple API for creating and 
populating a `ServiceLocator` with services found in these files. This is where the second dependency we declared
earlier comes into play.  
The HK2 Metadata Generator will generate hk2 inhabitant files during the compilation of your java files. It is a 
`JSR-269` annotation processor that handles the `@Service` annotation. The only requirement for using it is to put 
the `javax.inject`, `hk2-utils`, `hk2-api` and `hk2-metadata-generator` libraries in the classpath of the `javac`
process.

In this post we use the `HK2 Metadata Generator` in a Maven based build system:

{% highlight xml %}
<dependency>
  <groupId>org.glassfish.hk2</groupId>
  <artifactId>hk2-metadata-generator</artifactId>
</dependency>
{% endhighlight %}

Since Maven uses transitive dependencies this is all you need to add as a dependency during build.

In order to have our program automatically load the files generated with the hk2-inhabitant-generator you can use the 
`createAndPopulateServiceLocator` method near the start of our main method, like this:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/gs/Application.java' %}

{% highlight java %}
public class Application {

  public static void main(String... args) {
    ServiceLocator locator = ServiceLocatorUtilities.createAndPopulateServiceLocator();
    MessagePrinter messagePrinter = locator.getService(MessagePrinter.class);

    messagePrinter.printMessage();
  }
}
{% endhighlight %}

Just like that, we are done

# Conclusion
In this post we learned how to implement Dependency Injection using HK2. We demonstrated our example using services
and contracts and constructor based injection.  
You can find the source to this guide {% include source.html %}. Until the next post, keep doing cool things :smile:.


[HK2]:                          https://javaee.github.io/hk2/
[JDK]:                          http://www.oracle.com/technetwork/java/javase/downloads/index.html
[JSR-330]:                      https://www.jcp.org/en/jsr/detail?id=330
[Maven]:                        http://maven.apache.org/download.cgi