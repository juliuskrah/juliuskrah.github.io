---
layout:     post
title:      Auditing with Spring Data JPA
categories: blog
tags:       java maven jpa spring-data spring-boot reactive
section:    blog
author:     juliuskrah
repo:       spring-data-audit/tree/spring-data-jpa-audit
---
> [Spring Data][]{:target="_blank"} provides sophisticated support to transparently keep track of who created or
  changed an entity and the point in time this happened. 
  
# Introduction
Spring Data provides an abstraction layer to easily create a data access layer to a datastore. Check out the
[CRUD Operations with Spring Boot]({% post_url 2017-04-23-crud-operations-with-spring-boot %}) for an 
introduction to Spring Data.
In this post we will learn how to apply `Entity Auditing` using Spring Data. We will be using `@CreatedBy`, 
`@LastModifiedBy` to capture the user who created or modified the entity as well as `@CreatedDate` and 
`@LastModifiedDate` to capture the point in time this happened.

## Prerequisites
- [Java Development Kit][JDK]{:target="_blank"}  
- [Maven][]{:target="_blank"}

## Project Structure
At the end of this guide our folder structure will look similar to the following:

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__juliuskrah/
|  |  |  |  |  |__audit/
|  |  |  |  |  |  |__Application.java
|  |  |__resources/
|  |  |  |__db/
|  |  |  |  |__changelog/
|  |  |  |  |  |__db.changelog-master.yaml
|  |  |  |__application.yaml
|  |__test/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__juliuskrah/
|  |  |  |  |  |__audit/
|  |  |  |  |  |  |__ApplicationTests.java
|__pom.xml
```

## What You Need to Get Started
To help readers be able to follow along and do hands-on, I have provided an initial code that you can download as
[zip](https://github.com/juliuskrah/spring-data-audit/archive/v1.0.zip)/[tar.gz](https://github.com/juliuskrah/spring-data-audit/archive/v1.0.tar.gz).
Go ahead download and extract, and import into your favorite IDE as a maven project. Run and confirm everything 
works.

# Create the Required Auditing Classes
The first order of business is to create an Entity class and add the Spring Data auditing metadata:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/audit/Customer.java' %}

{% highlight java %}
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Customer implements Serializable {

    @Id
    @GeneratedValue(generator = "uuid2")
    @GenericGenerator(name = "uuid2", strategy = "uuid2")
    private String id;
    private String name;
    private String email;
    private String order;
    @CreatedBy                      // Audit metadata
    @Column(updatable = false)
    private String createdBy;
    @CreatedDate                    // Audit metadata
    @Column(updatable = false)
    private LocalDateTime createdDate;
    @LastModifiedBy                 // Audit metadata
    private String lastModifiedBy;
    @LastModifiedDate               // Audit metadata
    private LocalDateTime lastModifiedDate;

    // Getters and Setters omitted for brevity
}
{% endhighlight %}

> `AuditingEntityListener` is a JPA entity listener to capture auditing information on persiting and updating 
  entities.

Activate the Spring Data auditing feature by annotating the main configuration class with `@EnableJpaAuditing`:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/audit/Application.java' %}

{% highlight java %}
@EnableJpaAuditing
@SpringBootApplication
public class Application {
    //
}
{% endhighlight %}

Let's add some `CrudRepository`:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/audit/CustomerRepository.java' %}

{% highlight java %}
public interface CustomerRepository extends CrudRepository<Customer, String> {}
{% endhighlight %}

In our case, we are using `@CreatedBy` and `@LastModifiedBy`, the auditing infrastructure somehow needs to become 
aware of the current principal. To do so, we implement the `AuditorAware<T>` SPI interface to tell the 
infrastructure who the current user or system interacting with the application is. The generic type `T` defines of 
what type the properties annotated with `@CreatedBy` or `@LastModifiedBy` have to be:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/audit/SpringSecurityAuditorAware.java' %}

{% highlight java %}
@Component
public class SpringSecurityAuditorAware implements AuditorAware<String> {

    @Override
    public Optional<String> getCurrentAuditor() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

        if (authentication == null || !authentication.isAuthenticated()) {
            return Optional.of("system");
        }

        return  Optional.ofNullable(
            ((Principal)authentication.getPrincipal()).getName());
    }
}
{% endhighlight %}

We need to register the `AuditorAware` implementation with the Auditing infrastructure:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/audit/Application.java' %}

{% highlight java %}
@EnableJpaAuditing(auditorAwareRef = "springSecurityAuditorAware")
@SpringBootApplication
public class Application {
    //
}
{% endhighlight %}

Take note of the `auditorAwareRef` attribute with value `springSecurityAuditorAware`. The 
`springSecurityAuditorAware` is the bean name of the `AuditorAware` implementation.

# Add Some Routes
This step is added to test the Principal injected for `@CreatedBy` and `@LastModifiedBy`. 


[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org
[Spring Data]:              https://docs.spring.io/spring-data/commons/docs/current/reference/html/
