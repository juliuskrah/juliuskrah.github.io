---
layout:     post
title:      'CRUD operations with Hibernate and JPA'
categories: tutorial
tags:       java maven hibernate jpa
section:    series
author:     juliuskrah
repo:       java-crud/tree/hibernate-jpa
---
> [Hibernate ORM][Hibernate]{:target="_blank"} (**Hibernate** in short) is an [object-relational mapping][ORM]{:target="_blank"} 
  tool for the Java programming language. It provides a framework for mapping an object-oriented domain model to a relational database.  
  The [Java Persistence API][JPA]{:target="_blank"} (JPA)  provides Java developers with an object/relational mapping facility for 
  managing relational data in Java applications.
  
# Introduction
This is the first part of a series of posts focused on `Hibernate` and `JPA`. In this tutorial we are going to look at the basics of 
Hibernate and JPA.  
The relationship between Hibernate and JPA is that **Hibernate** is an implementation of the JPA specification. In this post we will create a
simple project that performs [CRUD][]{:target="_blank"} operations to demonstrates the powerful capabilities of JPA using Hibernate.

## Prerequisites
- [Java Development Kit][JDK]{:target="_blank"}  
- [Maven][]{:target="_blank"}

## Project Structure
We will create a [Maven][]{:target="_blank"} project using the standard directory structure as illustrated below:

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
Getting the dependencies to start using Hibernate and JPA is just a matter of adding the following to our `pom.xml` file:

{% highlight xml %}
<dependencies>
  <dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>5.2.6.Final</version>
  </dependency>
  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>1.4.193</version>
  </dependency>
</dependencies>
{% endhighlight %}

With this done, we can start writing some code.

# Setting up JPA
We will setup our `peristence unit` for JPA bootstraping. A `persistence.xml` file defines a persistence unit. The persistence.xml 
file is located in the `META-INF` directory of the root of the persistence unit. It may be used to specify managed persistence 
classes included in the persistence unit, object/relational mapping information for those classes, scripts for use in schema 
generation and the bulk loading of data, and other configuration information for the persistence unit and for the entity manager(s) 
and entity manager factory for the persistence unit.   
The object/relational mapping information can take the form of annotations on the managed persistence classes included in the 
persistence unit, an `orm.xml` file contained in the `META-INF` directory of the root of the persistence unit, one or more XML 
files on the classpath and referenced from the persistence.xml file, or a combination of these.

For our simple use case we will create a `persistence.xml` file.  
file: `src/main/resources/META-INF/persistence.xml`:

{% highlight xml %}
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
      <property name="hibernate.dialect" value="org.hibernate.dialect.H2Dialect" />
    </properties>
  </persistence-unit>
</persistence>
{% endhighlight %}

From the above snippet we have specified the persistence unit name as `com.juliuskrah.tutorial`. This is the value we will use to create
the [EntityManagerFactory][]{:target="_blank"}.  
In the same snippet we have specified the [Entity][]{:target="_blank"} class (`Person`) to be managed by the 
[EntityManager][]{:target="_blank"}  
We have also specified the properties Hibernate will use to connect to the database. The most important property here is the
`javax.persistence.provider` which tells JPA which implementation to use (in our case Hibernate).

The next thing to do is create the Entity class which maps to a database table.  
file: `src/main/java/com/tutorial/entity/Person.java`:

{% highlight java %}
@Entity
public class Person implements Serializable {

  @Id
  @GeneratedValue
  private Long id;

  private String firstName;

  private String lastName;

  // As of Hibernate 5.2 LocalDate is automatically mapped to sql type Date
  private LocalDate dateOfBirth;

  // As of Hibernate 5.2 LocalDateTime is automatically mapped to sql type Timestamp
  private LocalDateTime createdDate;

  private LocalDateTime modifiedDate;
 
  // Getters and Setters omitted for brevity

}
{% endhighlight %}

The `@Entity` annotation on the `Person` class is a mapping metadata that tells JPA to map the fields of the Person object to the columns
in the PERSON table. For example the `firstName` field of type `String` is mapped to `FIRSTNAME` column of type `varchar`. You can 
read more on Entity mapping [here][Domain Mapping]{:target="_blank"}.  
The `@Id` annotation on `id` field marks it as the `primary key` of the PERSON table. The `@GeneratedValue` annotation defines a 
strategy for generating primary key values (e.g. auto-increment).

## Managing Entities
With our entity set up we need a way to manage its lifecycle. Entities are managed by the entity manager, which is represented by 
`javax.persistence.EntityManager` instances. Each `EntityManager` instance is associated with a persistence context: a set of 
managed entity instances that exist in a particular data store. A persistence context defines the scope under which particular 
entity instances are created, persisted, and removed. The `EntityManager` interface defines the methods that are used to interact 
with the persistence context.

The `EntityManager` API creates and removes persistent entity instances, finds entities by the entity's primary key, and allows 
queries to be run on entities.  
The JPA specification defines two ways of managing entities: `Container-Managed Entity Managers` and `Application-Managed Entity Managers`.
This is beyond the scope of this tutorial. For the purposes of simplification, we will use an Application-Managed Entity Manager.

We will create a PersonRepository to facilitate our CRUD operations.  
file: `src/main/java/com/tutorial/repository/PersonRepositoryImpl.java`:

{% highlight java %}
public class PersonRepositoryImpl implements PersonRepository {
  private EntityManagerFactory emf = Persistence.createEntityManagerFactory("com.juliuskrah.tutorial");
  private EntityManager em;

  public PersonRepositoryImpl() {
    em = emf.createEntityManager();
  }

  @Override
  public Person create(Person person) {
    em.getTransaction().begin();
    em.persist(person);
    em.getTransaction().commit();
    return person;
  }

  @Override
  public Person read(Long id) {
    em.getTransaction().begin();
    Person person = em.find(Person.class, id);
    em.getTransaction().commit();
    return person;
  }

  @Override
  public Person update(Person person) {
    em.getTransaction().begin();
    person = em.merge(person);
    em.getTransaction().commit();
    return person;
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

At the top of the repository class is:

{% highlight java %}
private EntityManagerFactory emf = Persistence.createEntityManagerFactory("com.juliuskrah.tutorial");
{% endhighlight %}

The `createEntityManagerFactory` method takes as parameter `persistence unit name` which you will recall was set in our `persistence.xml`
file as `com.juliuskrah.tutorial`. The EntityManagerFactory is a [factory method][Factory]{:target="_blank"} used to create instances
of an EntityManager. The PersonRepository contains methods to perform the CRUD operations.

# Running the Application
We will create a main class to test our CRUD methods.

> **NOTE**  
  In our `persistence.xml` file we set a property `javax.persistence.schema-generation.database.action` to `drop-and-create`. This 
  instructs Hibernate to drop and recreate the database everytime the application is run. Please don't use this setting in a production
  environment.

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
      person.setFirstName("John");
      person.setLastName("Doe");
      person.setCreatedDate(LocalDateTime.now());
      person.setDateOfBirth(LocalDate.of(1993, Month.APRIL, 12));

      repository = new PersonRepositoryImpl();
      // Create person
      repository.create(person);

      person = null;
      // Hibernate creates a person proxy with an increment id of 1
      person = repository.read(1L);

      person.setModifiedDate(LocalDateTime.now());
      person.setFirstName("Jane");
      // Update person record
      repository.update(person);

      person = null;
      // Read updated record
      person = repository.read(1L);

      // Delete person
      repository.delete(person);

      // This will return null
      person = repository.read(1L);

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

The first thing to do here is start the H2 embedded database server:

{% highlight java %}
server = Server.createTcpServer().start();
{% endhighlight %}

Once the database server is up Hibernate can perform [DDL][]{:target="_blank"} and [DML][]{:target="_blank"} operations on the embedded
database server. We do a cleanup of resources when our application exits by calling `server.stop()` and `repository.close()` methods.

# Conclusion
In this post we were briefly introduced to JPA and its Hibernate implementation. We saw how to perform basic CRUD operations using 
the JPA APIs.  
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.



[Hibernate]: http://hibernate.org/orm/
[ORM]: https://en.wikipedia.org/wiki/Object-relational_mapping
[JPA]: https://docs.oracle.com/javaee/7/tutorial/partpersist.htm
[CRUD]: https://en.wikipedia.org/wiki/Create,_read,_update_and_delete
[Maven]: http://maven.apache.org
[JDK]: http://www.oracle.com/technetwork/java/javase/downloads/index.html
[EntityManagerFactory]: http://docs.oracle.com/javaee/7/api/javax/persistence/EntityManagerFactory.html
[Entity]: http://docs.oracle.com/javaee/7/api/javax/persistence/Entity.html
[EntityManager]: http://docs.oracle.com/javaee/7/api/javax/persistence/EntityManager.html
[Domain Mapping]: http://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html#domain-model
[Factory]: https://en.wikipedia.org/wiki/Factory_method_pattern
[DDL]: https://en.wikipedia.org/wiki/Data_definition_language
[DML]: https://en.wikipedia.org/wiki/Data_manipulation_language