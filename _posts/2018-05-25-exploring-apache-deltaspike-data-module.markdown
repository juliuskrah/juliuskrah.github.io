---
layout:     post
title:      Exploring Apache DeltaSpike Data Module
date:       2018-05-25 16:51:00 +0000
categories: blog
tags:       java maven javaee cdi jpa
section:    blog
author:     juliuskrah
repo:       cdi/tree/delta-spike
---
> [Apache DeltaSpike][DeltaSpike]{:target="_blank"} is a collection of portable 
  [CDI]({% post_url 2018-05-18-a-beginner-s-guide-to-cdi %}) extensions.

# Introduction

DeltaSpike Data Module provides capabilities for implementing repository patterns and thereby simplifying the 
repository layer not unlike [Spring Data](/tag/spring-data). In this post we will take a JPA project and 
rewrite the repository layer to use DeltaSpike Data. Basically, we will strip off the boilerplate 
`EntityManager` queries and enable centralization of query logic and consequently reducing code duplication 
and improve testability.

# Prerequisites

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
|  |  |  |  |  |__cdi/
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
|  |  |  |  |__templates/
|  |  |  |  |  |__default.xhtml
|  |  |  |  |__beans.xml
|  |  |  |  |__web.xml
|  |  |  |__index.html
|  |  |  |__index.xhtml
|__pom.xml
```

## Setting Up

Download the initial project ([zip](https://github.com/juliuskrah/cdi/archive/v2.0.zip)|
[tar.gz](https://github.com/juliuskrah/cdi/archive/v2.0.tar.gz)) and extract. From the extracted directory
run the following command:

```posh
C:\> mvn clean wildfly-swarm:run
```

Wait for all the dependencies to download and `swarm` to start.

# DeltaSpike

Add the DeltaSpike Data dependencies:

file: {% include file-path.html file_path='pom.xml' %}

{% highlight xml %}
...
<dependency>
  <groupId>org.apache.deltaspike.modules</groupId>
  <artifactId>deltaspike-data-module-api</artifactId>
  <version>1.8.1</version>
  <scope>compile</scope>
</dependency>
<dependency>
  <groupId>org.apache.deltaspike.modules</groupId>
  <artifactId>deltaspike-data-module-impl</artifactId>
  <version>1.8.1</version>
  <scope>runtime</scope>
</dependency>
...
{% endhighlight %}

Replace the code in `CustomerRepository`:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/cdi/repository/CustomerRepository.java' %}

{% highlight java %}
@Repository
public interface CustomerRepository extends EntityRepository<Customer, UUID> {

  @Query("SELECT c FROM Customer c WHERE c.id = :id")
  Optional<Customer> findOne(@QueryParam("id") UUID id);
}
{% endhighlight %}

There are a few things to note here. The `@Repository` annotation tells the extension that this is a 
repository for the `Customer` entity. Any method defined on the repository will be processed by the framework.

The `EntityRepository` Interface is mainly intended to hold complex query logic, working with both a 
repository and an `EntityManager` in the service layer might unnecessarily clutter code. 
The top base type is the `EntityRepository` interface, providing common methods used with an `EntityManager`.

DeltaSpike Data module supports also annotating methods for more control on the generated query using 
`@Query`. If the JPQL query requires named parameters to be used, this can be done by annotating the
arguments with the `@QueryParam` annotation.

For all these to work, DeltaSpike Data requires an `EntityManager` exposed via a CDI producer - which is 
common practice in Java EE applications:

```java
@Produces
@PersistenceContext
EntityManager em;
```

## Transactions

DeltaSpike Data module uses a `ResourceLocalTransactionStrategy` as default `TransactionStrategy` when 
demarcating transactions. Our example is configured to use `jta-data-source`, and the 
`ResourceLocalTransactionStrategy` does not quite fit our needs here.

Let's talk a little about JPA transactions to understand our decision not to use `ResourceLocalTransactionStrategy`.
A transaction is a set of operations that either fail or succeed as a unit. Transactions are a fundamental
part of persistence. A database transaction consists of a set of SQL DML (Data Manipulation Language) 
operations that are committed or rolled back as a single unit. An object level transaction is one in which a
set of changes made to a set of objects are committed to the database as a single unit.

JPA provides two mechanisms for transactions. When used in Java EE JPA provides integration with JTA (Java
Transaction API). JPA also provides its own `EntityTransaction` implementation for Java SE and for use in a
non-managed mode in Java EE. Transactions in JPA are always at the object level, this means that all changes 
made to all persistent objects in the persistence context are part of the transaction.

Resource local transactions are used in Java SE, or in application managed (non-managed) mode in Java EE. To 
use resource local transactions the transaction-type attribute in the `persistence.xml` is set to 
`RESOURCE_LOCAL`. Local JPA transactions are defined through the `EntityTransaction` class. It contains basic 
transaction API including `begin`, `commit` and `rollback`.

JTA transactions are used in Java EE, in managed mode (CMT). To use JTA transactions the transaction-type 
attribute in the `persistence.xml` is set to `JTA`. JTA transactions are defined through the JTA 
`UserTransaction` class.

JTA transactions can be used in two modes in Java EE. In Java EE managed mode such as an `EntityManager` 
injected through `@PersistenceContext`, this mode is also known as Container Managed Transaction (CMT).

The second mode allows the `EntityManager` to be application managed, (normally obtained from an injected 
EntityManagerFactory, or directly from JPA Persistence) also known as Bean Managed Transaction (BMT). This 
allows the persistence context to survive transaction boundaries, and follow the normal `EntityManager`
life-cycle similar to resource local.

DeltaSpike has three implementations of `TransactionStrategy`:

- `ResourceLocalTransactionStrategy` which uses `EntityTransaction`
- `BeanManagedUserTransactionStrategy` which uses `UserTransaction`
- `ContainerManagedTransactionStrategy`

We have to tell DeltaSpike to use `ContainerManagedTransactionStrategy`:

file: {% include file-path.html file_path='src/main/resources/META-INF/apache-deltaspike.properties' %}

{% highlight properties %}
globalAlternatives.org.apache.deltaspike.jpa.spi.transaction.TransactionStrategy = org.apache.deltaspike.jpa.impl.transaction.ContainerManagedTransactionStrategy
{% endhighlight %}

# Conclusion

We have been introduced to DeltaSpike Data Module, and we saw how easy it is to use for contructing queries.
We also did a quick introduction to Transactions in JPA. 

As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep 
doing cool things :+1:.

[DeltaSpike]:               https://deltaspike.apache.org/
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org