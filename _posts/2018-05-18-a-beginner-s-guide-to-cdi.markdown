---
layout:     post
title:      A Beginner's Guide to CDI
date:       2018-05-18 00:27:12 +0000
categories: blog
tags:       java maven javaee cdi rest jax-rs jpa
section:    blog
author:     juliuskrah
repo:       cdi/tree/2018.5.0
---
> [Contexts and Dependency Injection for the Java EE Platform][CDI]{:target="_blank"} is a 
  [JCP][]{:target="_blank"} standard for dependency injection and contextual lifecycle management and one of 
  the most important and popular parts of [Java EE](/tag/javaee/).

# Introduction

Contexts and Dependency Injection for Java EE (CDI) was introduced as part of the Java EE platform, and 
has quickly become one of the most important and popular components of the platform.

The CDI specification defines a set of complementary services that help improve the structure of application 
code. CDI layers an enhanced lifecycle and interaction model over existing Java component types, including 
managed beans and `Enterprise Java Beans`. 

You can run CDI in a Java EE or SE environment. However to leverage the full capabilities of CDI, it is 
recommended to use a Java EE container. In this tutorial, we will cover just the EE approach and we will be
using `Wildfly-Swarm`, a lighweight alternative to the Wildfly Java EE container which is also a very good
fit for deploying microservices.

## What We will Cover

CDI is quite broad; as it provides integration and contracts with the wider Java EE ecosystem. This being an 
introductory post, we limit the scope of what is covered to:

- Bean Scope
- Producer
- Inject
- Event

# Prerequisites

- [Java Development Kit][JDK]{:target="_blank"}
- [Maven][]{:target="_blank"}
- [Eclipse][]{:target="_blank"} 4.6.0+

## Project Structure

At the end of this guide our folder structure will look similar to the following:

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__juliuskrah/
|  |  |  |  |  |__cdi/
|  |  |  |  |  |  |__ApplicationContext.java
|  |  |  |  |  |  |__ApplicationResources.java
|  |  |  |  |  |  |__business/
|  |  |  |  |  |  |  |__CustomerService.java
|  |  |  |  |  |  |  |__dto/
|  |  |  |  |  |  |  |  |__CustomerBean.java
|  |  |  |  |  |  |  |__mapper/
|  |  |  |  |  |  |  |  |__CustomerMapper.java
|  |  |  |  |  |  |__entity/
|  |  |  |  |  |  |  |__Customer.java
|  |  |  |  |  |  |__repository/
|  |  |  |  |  |  |  |__CustomerRepository.java
|  |  |  |  |  |  |__web/
|  |  |  |  |  |  |  |__IndexController.java
|  |  |__resources/
|  |  |  |__db/
|  |  |  |  |__migration/
|  |  |  |  |  |__`V1__Create_customer_table.sql`
|  |  |  |__META-INF/
|  |  |  |  |__persistence.xml
|  |  |  |__modules/
|  |  |  |  |__com/
|  |  |  |  |  |__h2database/
|  |  |  |  |  |  |__h2/
|  |  |  |  |  |  |  |__main/
|  |  |  |  |  |  |  |  |__module.xml
|  |  |  |__project-defaults.yaml
|  |  |__webapp/
|  |  |  |__WEB-INF/
|  |  |  |  |__views/
|  |  |  |  |  |__index.xhtml
|  |  |  |  |  |__templates/
|  |  |  |  |  |  |__default.xhtml
|__pom.xml
```

## Setting Up

Download the initial project ([zip](https://github.com/juliuskrah/cdi/archive/v1.0.zip)|
[tar.gz](https://github.com/juliuskrah/cdi/archive/v1.0.tar.gz)) and extract.
Import the extracted project into Eclipse and follow the [Mapstruct instructions][Mapstruct]{:target="_blank"}
to generate mappers in Eclipse.

Run the application as a Java application; use `org.wildfly.swarm.Swarm` as the main class. ince it is 
up, access the application in your browser <http://localhost:8080>. Let's start by understanding what a bean
scope is.

## Bean Scopes

A bean is exactly what you think it is. Only now, it has a true identity in the container environment.

Managed Beans are defined as container-managed objects with minimal programming restrictions, otherwise known 
by the acronym POJO (Plain Old Java Object). They support a small set of basic services, such as resource 
injection, lifecycle callbacks and interceptors. Companion specifications, such as EJB and CDI, build on this 
basic model.

A bean is usually an application class that contains business logic. It may be called directly from Java code,
or it may be invoked via the Unified EL. A bean may access transactional resources. Dependencies between 
beans are managed automatically by the container. Most beans are stateful and contextual. The lifecycle of a 
bean is managed by the container.

The scope of a bean determines:

- the lifecycle of each instance of the bean and
- which clients share a reference to a particular instance of the bean.

For a given thread in a CDI application, there may be an active context associated with the scope of the 
bean. This context may be unique to the thread (for example, if the bean is request scoped), or it may be 
shared with certain other threads (for example, if the bean is session scoped) or even all other threads (if 
it is application scoped).

The scope of a bean defines the lifecycle and visibility of its instances. The CDI context model is 
extensible, accommodating arbitrary scopes. However, certain important scopes are built into the 
specification, and provided by the container. Each scope is represented by an annotation type.

According to the CDI specification, a scope determines:

- When a new instance of any bean with that scope is created
- When an existing instance of any bean with that scope is destroyed
- Which injected references refer to any instance of a bean with that scope

For example, if we have a session-scoped bean, `CustomerRepository` , all beans that are called in the 
context of the same `HttpSession` will see the same instance of `CustomerRepository`. This instance will be 
automatically created the first time a `CustomerRepository` is needed in that session, and automatically
destroyed when the session ends.

### Built-in Scopes

CDI defines four built-in scopes:

- `@RequestScoped`: A bean instance is created for each request.
- `@SessionScoped`: Creates a bean bound to the HTTP Session.
- `@ApplicationScoped`: A single bean instance is shared by the application.
- `@ConversationScoped`: The conversation scope is a bit like the traditional session scope in that it holds
   state associated with a user of the system, and spans multiple requests to the server. 

   A conversation represents a task—a unit of work from the point of view of the user. The conversation 
   context holds state associated with what the user is currently working on. If the user is doing multiple 
   things at the same time, there are multiple conversations.

   The conversation context is active during any servlet request (since CDI 1.1). Most conversations are 
   destroyed at the end of the request. If a conversation should hold state across multiple requests, it must 
   be explicitly promoted to a long-running conversation.

Let's start by creating some Application-Scoped beans:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/cdi/repository/CustomerRepository.java' %}

{% highlight java %}
@ApplicationScoped
public class CustomerRepository {
  // ...
}
{% endhighlight %}

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/cdi/business/CustomerService.java' %}

{% highlight java %}
@ApplicationScoped
public class CustomerService {
  // ...
}
{% endhighlight %}

We can `Inject` these beans into other beans. We will cover this later in this post.

## Producers

CDI distinguishes between  two types of producers:

- Producer methods let us overcome certain limitations that arise when a container, instead of the 
  application, is responsible for instantiating objects. They’re also the easiest way to integrate objects 
  which are not beans into the CDI environment.
- A producer field is a simpler alternative to a producer method. A producer field is declared by annotating 
  a field of a bean class with the `@Produces` annotation — the same annotation used for producer methods.
  A producer field is really just a shortcut that lets us avoid writing a useless getter method.

Let's demonstrate that with an example:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/cdi/ApplicationResources.java' %}

{% highlight java %}
@RequestScoped
public class ApplicationResources {

  @Produces
  @PersistenceContext
  EntityManager em;

  @Produces
  public Logger produceLog(InjectionPoint injectionPoint) {
    return Logger.getLogger(injectionPoint.getMember().getDeclaringClass());
  }
}
{% endhighlight %}

We created a producer field `EntityManager`, which will enable us `inject` an EntityManager into CDI beans.
We also created a producer method `Logger` to easily inject a logger into bean classes.

## Inject

The `@Inject` annotation lets us define an injection point that is injected during bean instantiation. 
Injection can occur via three different mechanisms.

- _Bean constructor_ parameter injection: A bean can only have one injectable constructor.
- _Initializer method_ parameter injection: A bean can have multiple initializer methods.
- And direct field injection: Getter and setter methods are not required for field injection to work.

Dependency injection always occurs when the bean instance is first instantiated by the container. Simplifying
just a little, things happen in this order:

- First, the container calls the bean constructor (the default constructor or the one annotated `@Inject`), 
  to obtain an instance of the bean.
- Next, the container initializes the values of all injected fields of the bean.
- Next, the container calls all initializer methods of bean (the call order is not portable, don’t rely on it).
- Finally, the `@PostConstruct` method, if any, is called.

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/cdi/repository/CustomerRepository.java' %}

{% highlight java %}
@ApplicationScoped
public class CustomerRepository {
  private EntityManager em;

  public CustomerRepository() {}

  @Inject
  public CustomerRepository(EntityManager em) {
    this.em = em;
  }

  // ...
}
{% endhighlight %}

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/cdi/web/IndexController.java' %}

{% highlight java %}
@Path("/")
@Controller
public class IndexController {
  @Inject
  private CustomerService customerService;
  @Inject
  private Logger log;

  // ...
}
{% endhighlight %}

## Events

Dependency injection enables loose-coupling by allowing the implementation of the injected bean type to vary,
either at deployment time or runtime. Events go one step further, allowing beans to interact with no compile 
time dependency at all. Event producers raise events that are delivered to event observers by the container.

This basic schema might sound like the familiar observer/observable pattern, but there are a couple of twists:

- not only are event producers decoupled from observers; observers are completely decoupled from producers,
- observers can specify a combination of "selectors" to narrow the set of event notifications they will 
  receive, and
- observers can be notified immediately, or can specify that delivery of the event should be delayed until 
  the end of the current transaction.

CDI events are made up of:

- Event Payload: The event object carries state from producer to consumer. The event object is nothing more 
  than an instance of a concrete Java class. (The only restriction is that an event type may not contain type 
  variables).
- Event Observers: An observer method is a method of a bean with a parameter annotated `@Observes` or 
  `@ObservesAsync`. The annotated parameter is called the _event parameter_. The type of the event parameter
  is the observed _event type_.
- Event Producers: Event producers fire events either synchronously or asynchronously using an instance of 
  the parameterized `Event` interface. An instance of this interface is obtained by injection.

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/cdi/business/CustomerService.java' %}

{% highlight java %}
@ApplicationScoped
public class CustomerService {
  @Inject // Inject event
  private Event<CustomerBean> customerEventSrc;

  public CustomerBean createCustomer(CustomerBean customer) {
    Customer c = // ...
    // Produce customer event
    customerEventSrc.fire(customer);
    return customer;
  }

  /**
   * The producer and observer do not necessarily have to be in the same class
   */
  public void onCustomerListChanged(
    // Observe customer event 
    @Observes(notifyObserver = Reception.IF_EXISTS, 
    during = TransactionPhase.AFTER_COMPLETION) 
    final CustomerBean customer) {
    // Add your logic here
  }
  // ...
}
{% endhighlight %}

# Conclusion

This is an introductory post to CDI. We covered the basics of CDI with `@Inject`, `@Produces` and bean scopes.
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep 
doing cool things :+1:.

[CDI]:                      http://cdi-spec.org/
[Eclipse]:                  https://www.eclipse.org/downloads/
[JCP]:                      https://jcp.org/en/home/index
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Mapstruct]:                http://mapstruct.org/documentation/ide-support/
[Maven]:                    http://maven.apache.org
