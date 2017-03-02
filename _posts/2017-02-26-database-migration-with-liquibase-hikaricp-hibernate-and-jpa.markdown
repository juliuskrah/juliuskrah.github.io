---
layout:     series
title:      'Database Migration with Liquibase, HikariCP, Hibernate and JPA'
date:       2017-02-26 16:19:59 +0000
categories: tutorial
tags:       java maven hibernate jpa hikaricp liquibase
section:    series
author:     juliuskrah
repo:       java-crud/tree/liquibase-hibernate-jpa
---
> [`Liquibase`][Liquibase]{:target="_blank"} is source control for your database. `Liquibase` is an open source database-independent 
  library for tracking, managing and applying database schema changes.

# Introduction
This is the third [part]({% post_url 2017-02-15-crud-operations-with-hibernate-and-jpa %}) of a [series]({% post_url 2017-02-16-getting-started-with-hikaricp-hibernate-and-jpa %})
of posts focused on `Hibernate` and `JPA`.  In this tutorial we are going to look at database migrations with `Liquibase`.
When implementing and deploying a new version of an application, simple and fast refactoring of your database model is one of the most
important things in order to implement flexible business requirements. Liquibase supports tracking, managing and applying database 
schema changes.

## Prerequisites
- [Java Development Kit][JDK]{:target="_blank"}  
- [Maven][]{:target="_blank"}

## Project Structure
This is a build up on previous posts and our folder structure will remain relatively the same:

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
|  |  |  |__dbChangelog.xml
|  |  |  |__log4j2.properties
|__pom.xml
```

# Setting up Dependencies
To add the Liquibase dependency to the project add the following dependency by modifying the `pom.xml` file:

{% highlight xml %}
<dependencies>
  <dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
    <version>3.5.3</version>
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
        value="none" />
      <property name="javax.persistence.provider"
        value="org.hibernate.jpa.HibernatePersistenceProvider" />
      <property name="javax.persistence.jdbc.driver" value="org.h2.Driver" />
      <property name="javax.persistence.jdbc.url" value="jdbc:h2:file:./target/test;DB_CLOSE_DELAY=-1;MVCC=true" />
      <property name="javax.persistence.jdbc.user" value="sa" />
      <property name="javax.persistence.jdbc.password" value="" />
      <property name="hibernate.hikari.minimumIdle" value="5" />
      <property name="hibernate.hikari.maximumPoolSize" value="10" />
      <property name="hibernate.hikari.idleTimeout" value="30000" />
      <property name="hibernate.connection.handling_mode" value="delayed_acquisition_and_release_after_transaction" />
      <property name="hibernate.connection.provider_class" value="org.hibernate.hikaricp.internal.HikariCPConnectionProvider" />
    </properties>
  </persistence-unit>
</persistence>
{% endhighlight %}

The following properties have been modified in the `persistence.xml` file

- `javax.persistence.schema-generation.database.action` value set to `none` (Liquibase will take care of creating database)
- `javax.persistence.jdbc.url` value set to `jdbc:h2:file:./target/test;DB_CLOSE_DELAY=-1;MVCC=true` (Will also work with previous value)
- `hibernate.connection.handling_mode` this property is added for performance reasons. This tells hibernate to release connection back
   into pool after use.

# Creating the Changelog File
Liquibase [database changelog][Changelog]{:target="_blank"} file is where all database changes are listed. Liquibase supports 
`XML`, `YAML`, `JSON` and `SQL` as formats for `Changelog` files. Beyond these built-in formats, the Liquibase extension system 
allows you to create changelog files in whatever format you like. This makes it highly flexible.  
For this tutorial we will use the `XML` changelog format: 

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
      <column name="firstName" type="varchar(255)"/>
      <column name="lastname" type="varchar(255)"/>
      <column name="dateOfBirth" type="date"/>
      <column name="createdDate" type="timestamp"/>
      <column name="modifiedDate" type="timestamp"/>
    </createTable>
  </changeSet>
</databaseChangeLog>
{% endhighlight %}

The above is self explanatory. The `createSequence` is needed so Hibernate knows how to autogenerate the primary key sequence.

Now that we have setup our changelog, we program our application to run migrations each time the application is started.

file: `src/main/java/com/tutorial/Application.java`:

{% highlight java linenos %}
...
private static void init(EntityManager em) {

  Connection connection = em.unwrap(SessionImpl.class).connection();
  Database database = null;
  Liquibase liquibase = null;

  try {
    database = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(new JdbcConnection(connection));
    liquibase = new Liquibase("dbChangelog.xml", new ClassLoaderResourceAccessor(), database);
    liquibase.update("test");
  } catch (LiquibaseException e) {
    e.printStackTrace();
  }
}
{% endhighlight %}

On `line 4` we get the connection created by the `EntityManager` and use it to create the Liquibase Database object. Next on `line 11`
we start the migration by calling the `update()` method.

# Putting it Together
Finally let us put it all together and run the application.

file: `src/main/java/com/tutorial/Application.java`:

{% highlight java %}
public class Application {

  public static void main(String[] args) {
    PersonRepositoryImpl repository = null;
    try {
      repository = new PersonRepositoryImpl();

      init(repository.getEntityManager());

      Person person = new Person();
      person.setFirstName("Julius");
      person.setLastName("Krah");
      person.setCreatedDate(LocalDateTime.now());
      person.setDateOfBirth(LocalDate.of(1990, Month.APRIL, 4));

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
        System.out.format("Person from database: %s", consumer);
      });
      // Delete person
      repository.delete(p.get());

      p = Optional.empty();

      p = repository.read(1L);
			
    } finally {
      if (repository != null)
        repository.close();
    }
  }

  private static void init(EntityManager em) {

    Connection connection = em.unwrap(SessionImpl.class).connection();
    Database database = null;
    Liquibase liquibase = null;

    try {
      database = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(new JdbcConnection(connection));
      liquibase = new Liquibase("dbChangelog.xml", new ClassLoaderResourceAccessor(), database);
      liquibase.update("test");
    } catch (LiquibaseException e) {
      e.printStackTrace();
    }
  }
}
{% endhighlight %}


# Conclusion
In this post we implemented database migrations using Liquibase. We created a database changelog file with our changes. We also learnt
how to run these changes programmatically.    
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.




[Liquibase]:    http://www.liquibase.org/
[Maven]:        http://maven.apache.org
[JDK]:          http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Changelog]:    http://www.liquibase.org/documentation/databasechangelog.html