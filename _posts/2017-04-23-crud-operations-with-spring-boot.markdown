---
layout:     series
title:      CRUD Operations with Spring Boot
date:       2017-04-23 21:02:33 +0000
categories: tutorial
tags:       java maven hibernate jpa spring-data spring-boot
section:    series
author:     juliuskrah
repo:       java-crud/tree/spring-boot-hibernate-jpa
---
> [Spring Boot][]{:target="_blank"} makes it easy to create stand-alone, production-grade Spring based Applications that you can 
  "just run". It is based on a model of `Convention over Configuration` and good intentions.

# Introduction
This is the sixth part part of a series of posts focused on `Hibernate` and `JPA`. In my previous post we demonstrated how to write
CRUD functions using interfaces in [Spring Data]({% post_url 2017-03-22-crud-operations-with-spring-data-jpa %}). In this post we will
look at reducing configuration with [Spring Boot][]{:target="_blank"} and writing less boilerplate code.   
At the end of this post you will have familiarised yourself with `Spring Boot`.

## Prerequisites
- [Java Development Kit][JDK]{:target="_blank"}  
- [Maven][]{:target="_blank"}

## Project Structure
Building up on previous posts, our folder structure will remain relatively the same:

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__tutorial/
|  |  |  |  |  |__Application.java
|  |  |  |  |  |__entity/
|  |  |  |  |  |  |__Person.java
|  |  |  |  |  |__repository/
|  |  |  |  |  |  |__PersonRepository.java
|  |  |__resources/
|  |  |  |__application.properties
|  |  |  |__dbChangelog.xml
|  |  |  |__log4j2.properties
|__pom.xml
```

# Setting up Dependencies
To set up a `Spring Boot` project we need the following dependencies

- spring-boot-starter

Modify the `pom.xml` file by making the following changes:

{% highlight xml %}
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>1.5.2.RELEASE</version>
  <relativePath />
</parent>

<properties>
  <hibernate.version>5.2.6.Final</hibernate.version><!-- Override version 5.0 used by Spring Boot -->
  <java.version>1.8</java.version>
</properties>

<dependencies>
  ...
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
    <exclusions>
      <exclusion><!-- Hibernate EntityManager was merged into Hibermate core in 5.2 -->
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-entitymanager</artifactId>
      </exclusion>
      <exclusion>
        <groupId>org.apache.tomcat</groupId><!-- We will be using HikariCP pooling library -->
        <artifactId>tomcat-jdbc</artifactId>
      </exclusion>
    </exclusions>
  </dependency>
</dependencies>

<build>
  <plugins>
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
    </plugin>
  </plugins>
</build>
{% endhighlight %}

Wow! That's a lot of changes. But that will be all for our `POM.xml`.

# Naming Strategy
Hibernate 5 deprecated the legacy [`NamingStrategy`][Legacy Naming]{:target="_blank"} in favour of 
[`ImplicitNamingStrategy` and `PhysicalNamingStrategy`][Naming]{:target="_blank"}. With the release of Spring Boot 1.4, a new 
[`SpringPhysicalNamingStrategy`][Spring Naming]{:target="_blank"} is `auto configured` to align with Hibernate 5.  
Part of the mapping of an object model to the relational database is mapping names from the object model to the corresponding database names. The `SpringPhysicalNamingStrategy` uses snake case for the database names. In this regard we will modify our
`Liquibase Changelog` file.

file: `src/main/resources/dbChangelog.xml`:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog 
  xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog 
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.5.xsd">
  <property name="autoIncrement" value="true" dbms="mysql,h2,postgresql,oracle,mssql"/>

  <changeSet id="0" author="julius" dbms="h2,postgresql,oracle">
    <createSequence sequenceName="hibernate_sequence" startValue="1" incrementBy="1"/>
  </changeSet>
		
  <changeSet id="1" author="julius">
    <comment>Create Person table</comment>
    <createTable tableName="person">
      <column name="id" type="bigint" autoIncrement="${autoIncrement}">
        <constraints primaryKey="true" nullable="false" />
      </column>
      <column name="first_name" type="varchar(255)"/>
      <column name="last_name" type="varchar(255)"/>
      <column name="date_of_birth" type="date"/>
      <column name="created_date" type="timestamp"/>
      <column name="modified_date" type="timestamp"/>
    </createTable>
  </changeSet>
</databaseChangeLog>
{% endhighlight %}

## Plumbing
We now made a few minor changes to get Spring Boot up and running.

file: `src/main/resources/application.properties`:

{% highlight properties %}
spring.jpa.hibernate.ddl-auto=none
spring.jpa.properties.javax.persistence.provider=org.hibernate.jpa.HibernatePersistenceProvider
spring.jpa.properties.hibernate.hikari.dataSourceClassName=org.h2.jdbcx.JdbcDataSource
spring.jpa.properties.hibernate.hikari.dataSource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;MVCC=true
spring.jpa.properties.hibernate.hikari.dataSource.user=sa
spring.jpa.properties.hibernate.hikari.dataSource.password=
spring.jpa.properties.hibernate.hikari.minimumIdle=5
spring.jpa.properties.hibernate.hikari.maximumPoolSize=10
spring.jpa.properties.hibernate.hikari.idleTimeout=30000
spring.jpa.properties.hibernate.connection.handling_mode=delayed_acquisition_and_hold
spring.jpa.properties.hibernate.connection.provider_class=org.hibernate.hikaricp.internal.HikariCPConnectionProvider

liquibase.change-log=classpath:/dbChangelog.xml
liquibase.check-change-log-location=true
liquibase.contexts=test
liquibase.drop-first=true
liquibase.enabled=true
{% endhighlight %}

# Spring Boot
Spring Boot is all about reducing the amount of boilerplate code you have to write.  
Remove:

file: `src/main/java/com/tutorial/Application.java`:
{% highlight java %}
@Configuration
@EnableJpaRepositories
@EnableTransactionManagement
@PropertySource("classpath:application.properties")
{% endhighlight %}

replace with:

{% highlight java %}
@SpringBootApplication
{% endhighlight %}

Next remove all beans from the `Application` class and we end up with this:

{% highlight java %}
@SpringBootApplication
public class Application {

  public static void main(String[] args) {
    ApplicationContext ctx = SpringApplication.run(Application.class, args);
		
    PersonRepository repository = ctx.getBean(PersonRepository.class);
    Person person = new Person();
    person.setFirstName("Julius");
    person.setLastName("Krah");
    person.setCreatedDate(LocalDateTime.now());
    person.setDateOfBirth(LocalDate.of(1990, Month.APRIL, 4));

    // Create person
    repository.save(person);

    // Hibernate generates id of 1
    Optional<Person> p = repository.findOne(1L);

    p.ifPresent(consumer -> {
      consumer.setModifiedDate(LocalDateTime.now());
      consumer.setFirstName("Abeiku");
    });
    // Update person record
    repository.save(p.get());

    p = Optional.empty();

    // Read updated record
    p = repository.findOne(1L);
    p.ifPresent(consumer -> {
      System.out.format("Person updated: %s", consumer);
    });
    // Delete person
    repository.delete(p.get());

    p = Optional.empty();

    p = repository.findOne(1L);
  }
}
{% endhighlight %}

That's it.

# Conclusion
In this post we did a rewrite of our previous articles in the series to use Spring Boot. We also saw how easy it is to configure a 
Spring Boot project.
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.


[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org
[Spring Boot]:              http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/
[Naming]:                   http://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html#naming
[Legacy Naming]:            http://docs.jboss.org/hibernate/orm/current/javadocs/org/hibernate/cfg/NamingStrategy.html
[Spring Naming]:            https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-1.4-Release-Notes#hibernate-5
