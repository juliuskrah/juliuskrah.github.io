---
layout:     post
title:      'Getting Started with HikariCP, Hibernate and JPA'
categories: tutorial
tags:       java maven hibernate jpa hikaricp
section:    series
author:     juliuskrah
repo:       java-crud/tree/hikari-hibernate-jpa
---
> [HikariCP][]{:target="_blank"} is a “zero-overhead” production-quality connection pool.

# Introduction
This is the second part of a series of posts focused on `Hibernate` and `JPA`. In this tutorial we are going to look at database 
pooling using HikariCP for Hibernate and JPA.  
A connection pool is a cache of database connections maintained so that the connections can be reused when future requests to the 
database are required. Connecting to a data source can be time consuming. To minimize the cost of opening connections, you need to 
implement a pooling mechanism in your application.

## Prerequisites
- [Java Development Kit][JDK]{:target="_blank"}  
- [Maven][]{:target="_blank"}

## Project Structure
This is a follow up to the [previous]({% post_url 2017-02-15-crud-operations-with-hibernate-and-jpa %}) post. Our folder structure 
will remain the same:

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
|  |  |  |  |  |  |__PersonRepositoryImpl.java
|  |  |__resources/
|  |  |  |__META-INF/
|  |  |  |  |__persistence.xml
|  |  |  |__log4j2.properties
|__pom.xml
```

# Setting up Dependencies
To add the HikariCP dependency to the project replace the `hibernate-core` dependency with `hibernate-hikaricp` in our 
`pom.xml` file:

{% highlight xml %}
<dependencies>
  <dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-hikaricp</artifactId>
    <version>5.2.6.Final</version>
  </dependency>
  ...
</dependencies>
{% endhighlight %}

# JPA Configuration
The next change will be our `persistence.xml` file.  
file: `src/main/resources/META-INF/persistence.xml`:

{% highlight java %}
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="2.1"
    xmlns="http://xmlns.jcp.org/xml/ns/persistence" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence
    http://xmlns.jcp.org/xml/ns/persistence/persistence_2_1.xsd">
  <persistence-unit name="com.juliuskrah.tutorial"
    transaction-type="RESOURCE_LOCAL">
    <class>com.tutorial.entity.Person</class>
    <properties>
      <property name="javax.persistence.schema-generation.database.action"
        value="drop-and-create" />
      <property name="javax.persistence.provider"
        value="org.hibernate.jpa.HibernatePersistenceProvider" />
      <property name="javax.persistence.jdbc.driver" value="org.h2.Driver" />
      <property name="javax.persistence.jdbc.url" value="jdbc:h2:tcp://localhost/~/test" />
      <property name="javax.persistence.jdbc.user" value="sa" />
      <property name="javax.persistence.jdbc.password" value="" />
      <property name="hibernate.hikari.minimumIdle" value="5" />
      <property name="hibernate.hikari.maximumPoolSize" value="10" />
      <property name="hibernate.hikari.idleTimeout" value="30000" />
      <property name="hibernate.connection.provider_class" value="org.hibernate.hikaricp.internal.HikariCPConnectionProvider" />
    </properties>
  </persistence-unit>
</persistence>
{% endhighlight %}

We modified the `persistence.xml` file to have the following `HikariCP` specific configuration.

- `hibernate.hikari.minimumIdle`
- `hibernate.hikari.maximumPoolSize`
- `hibernate.hikari.idleTimeout`
- `hibernate.connection.provider_class`

We set the [pool size][Pool Sizing]{:target="_blank"} to `10`. In a production system you may set this value a little higher.

Another change from the original [post]({% post_url 2017-02-15-crud-operations-with-hibernate-and-jpa %}) is in the `PersonRepository` 
class. This change is trivial and does not impact HikariCP pooling in any way. It is just an assertive switch to 
`Java8`'s [`Optional`][Optional]{:target="_blank"} to better handle `null` values.

file: `src/main/java/com/tutorial/repository/PersonRepositoryImpl.java`:

{% highlight java %}
public class PersonRepositoryImpl implements PersonRepository {
  private EntityManagerFactory emf = Persistence.createEntityManagerFactory("com.juliuskrah.tutorial");
  private EntityManager em;

  public PersonRepositoryImpl() {
    em = emf.createEntityManager();
  }

  @Override
  public Optional<Person> create(Person person) {
    Objects.requireNonNull(person, "Person must not be null");
    em.getTransaction().begin();
    em.persist(person);
    em.getTransaction().commit();
    return Optional.of(person);
  }

  @Override
  public Optional<Person> read(Long id) {
    em.getTransaction().begin();
    Person person = em.find(Person.class, id);
    em.getTransaction().commit();
    return Optional.ofNullable(person);
  }

  @Override
  public Optional<Person> update(Person person) {
    Objects.requireNonNull(person, "Person must not be null");
    em.getTransaction().begin();
    person = em.merge(person);
    em.getTransaction().commit();
    return Optional.of(person);
  }

  @Override
  public void delete(Person person) {
    em.getTransaction().begin();
    em.remove(person);
    em.getTransaction().commit();
  }

  @Override
  public void close() {
    emf.close();
  }
}
{% endhighlight %}

# Starting the Application
With HikariCP setup in our application let us run the sample. Final piece to add is the updated `main` class.

file: `src/main/java/com/tutorial/Application.java`:

{% highlight java %}
public class Application {

  public static void main(String[] args) {
    Server server = null;
    PersonRepositoryImpl repository = null;
    try {
      // Start H2 embedded database
      server = Server.createTcpServer().start();

      Person person = new Person();
      person.setFirstName("Julius");
      person.setLastName("Krah");
      person.setCreatedDate(LocalDateTime.now());
      person.setDateOfBirth(LocalDate.of(1990, Month.APRIL, 4));

      repository = new PersonRepositoryImpl();
      // Create person
      repository.create(person);

      // Hibernate generates id of 1
      Optional<Person> p = repository.read(1L);

      p.ifPresent(consumer -> {
        consumer.setModifiedDate(LocalDateTime.now());
        consumer.setFirstName("Abeiku");
      });
      // Update person record
      repository.update(p.get());
			
      p = Optional.empty();

      // Read updated record
      p = repository.read(1L);
      p.ifPresent(consumer -> {
        System.out.format("Person updated: %s", consumer);
      });
      // Delete person
      repository.delete(p.get());
			
      p = Optional.empty();

      p = repository.read(1L);

    } catch (SQLException e) {
      e.printStackTrace();
    } finally {
      if (repository != null)
        repository.close();
      if (server != null)
        server.stop();
    }
  }
}
{% endhighlight %}

Just run the main method.

# Conclusion
In this post we understood the basics of setting up HikariCP with Hibernate and JPA with minimal configuration.  
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.


[HikariCP]: http://brettwooldridge.github.io/HikariCP/
[Pool Sizing]: https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing
[Optional]: http://docs.oracle.com/javase/8/docs/api/java/util/Optional.html
[Maven]: http://maven.apache.org
[JDK]: http://www.oracle.com/technetwork/java/javase/downloads/index.html
