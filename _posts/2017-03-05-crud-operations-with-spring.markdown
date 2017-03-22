---
layout:     series
title:      CRUD Operations with Spring
date:       2017-03-05 23:09:57 +0000
categories: tutorial
tags:       java maven hibernate jpa spring liquibase
section:    series
author:     juliuskrah
repo:       java-crud/tree/spring-hibernate-jpa
---
> The [Spring Framework][Spring]{:target="_blank"} is an application framework and inversion of control container for the Java 
  platform. The framework's core features can be used by any Java application, but there are extensions for building web 
  applications on top of the Java EE platform. Although the framework does not impose any specific programming model, it has 
  become popular in the Java community as an alternative to, replacement for, or even addition to the Enterprise JavaBeans 
  (EJB) model.

# Introduction
This is the fourth part of a series of posts focused on `Hibernate` and `JPA`. Throughout this series we have learnt the basics
of [JPA]({% post_url 2017-02-15-crud-operations-with-hibernate-and-jpa %}), setup pooling with [HikariCP]({% post_url 2017-02-16-getting-started-with-hikaricp-hibernate-and-jpa %})
and database migration with [Liquibase]({% post_url 2017-02-26-database-migration-with-liquibase-hikaricp-hibernate-and-jpa %}).  
In this post we will refactor the source code to take advantage of Spring's programming paradigm.

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
|  |  |  |__application.properties
|  |  |  |__dbChangelog.xml
|  |  |  |__log4j2.properties
|__pom.xml
```

# Setting up Dependencies
To setup the project in Spring we need the following dependecies

- spring-core
- spring-tx
- spring-orm
- spring-context

Modify the `pom.xml` file by adding the following dependencies:

{% highlight xml %}
<dependencies>
  ...
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>4.3.7.RELEASE</version>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-orm</artifactId>
    <version>4.3.7.RELEASE</version>
  </dependency>
</dependencies>
{% endhighlight %}

The above dependencies will transitively pull in all the other spring dependencies.

# Setting up Spring
The first thing we need to do is delete the `persistence.xml` file. We will configure the persistence unit programmatically using
Spring.  
Next is to declare `annotations` on the `Application` class to set it up as a Spring [configuration]({% post_url 2016-12-10-introduction-to-spring-and-dependency-injection-jsr-330 %}) 
class.

file: `src/main/java/com/tutorial/Application.java`:

{% highlight java %}
@Configuration
@ComponentScan
public class Application {...}
{% endhighlight %}

Define the `PersonRepositoryImpl` class as a Spring `Component` to be detected by the `@ComponentScan` annotation.

file: `src/main/java/com/tutorial/repository/PersonRepositoryImpl.java`:

{% highlight java %}
@Component
public class PersonRepositoryImpl implements PersonRepository {...}
{% endhighlight %}

With this done, let's bootstrap two beans leveraging Spring's JPA support.

file: `src/main/java/com/tutorial/Application.java`:

{% highlight java %}
...
@Bean
public LocalContainerEntityManagerFactoryBean entityManager() {
  LocalContainerEntityManagerFactoryBean entityManager = new LocalContainerEntityManagerFactoryBean();
  entityManager.setPackagesToScan("com.tutorial.entity");
  entityManager.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
  entityManager.setJpaProperties(properties());
  entityManager.setPersistenceUnitName("com.juliuskrah.tutorial");

  return entityManager;
}

@Bean
public PlatformTransactionManager transactionManager(EntityManagerFactory emf) {
  return new JpaTransactionManager(emf);
}

private Properties properties() {
  Properties props = new Properties();
  props.setProperty("javax.persistence.provider", "org.hibernate.jpa.HibernatePersistenceProvider");
  props.setProperty("javax.persistence.schema-generation.database.action", "none");
  props.setProperty("hibernate.hikari.dataSourceClassName", "org.h2.jdbcx.JdbcDataSource");
  props.setProperty("hibernate.hikari.dataSource.url", "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1;MVCC=true");
  props.setProperty("hibernate.hikari.dataSource.user", "sa");
  props.setProperty("hibernate.hikari.dataSource.password", "");
  props.setProperty("hibernate.hikari.minimumIdle", "5");
  props.setProperty("hibernate.hikari.maximumPoolSize", "10");
  props.setProperty("hibernate.hikari.idleTimeout", "30000");
  props.setProperty("hibernate.connection.handling_mode", "delayed_acquisition_and_hold");
  props.setProperty("hibernate.connection.provider_class", "org.hibernate.hikaricp.internal.HikariCPConnectionProvider");

  return props;
}
{% endhighlight %} 

With a transaction manager bean defined, we can add `@EnableTransactionManagement` annotation on the `Application` class.  
We have defined our `EntityManager` bean and will need to replace:

file: `src/main/java/com/tutorial/repository/PersonRepositoryImpl.java`:

{% highlight java %}
private EntityManagerFactory emf = Persistence.createEntityManagerFactory("com.juliuskrah.tutorial");
{% endhighlight %} 

with:

file: `src/main/java/com/tutorial/repository/PersonRepositoryImpl.java`:

{% highlight java %}
@PersistenceUnit(unitName = "com.juliuskrah.tutorial")
private EntityManagerFactory emf;
{% endhighlight %} 

We are almost done with the changes. Add `@Transactional` on the PersonRepositoryImpl` class and remove all transactional code.
Transaction management will be handled by the Spring container. The final class will look like this:

file: `src/main/java/com/tutorial/repository/PersonRepositoryImpl.java`:

{% highlight java %}
@Component
@Transactional
public class PersonRepositoryImpl implements PersonRepository {
  @PersistenceUnit(unitName = "com.juliuskrah.tutorial")
  private EntityManagerFactory emf;
  private EntityManager em;

  public PersonRepositoryImpl() {}

  @PostConstruct
  public void initEntityManager() {
    em = emf.createEntityManager();
  }

  @Override
  public Optional<Person> create(Person person) {
    Objects.requireNonNull(person, "Person must not be null");
    em.persist(person);
    return Optional.of(person);
  }

  @Transactional(readOnly = true)
  @Override
  public Optional<Person> read(Long id) {
    Person person = em.find(Person.class, id);
    return Optional.ofNullable(person);
  }

  @Override
  public Optional<Person> update(Person person) {
    Objects.requireNonNull(person, "Person must not be null");
    person = em.merge(person);
    return Optional.of(person);
  }

  @Override
  public void delete(Person person) {
    em.remove(person);
  }
}
{% endhighlight %}

# Setting up Migration
The last piece is to setup `Liquibase` migration using Spring. To achieve this we need to bootstrap two more beans:

file: `src/main/java/com/tutorial/Application.java`:

{% highlight java %}
...
@Bean(destroyMethod = "close")
public DataSource dataSource() {
  HikariConfig config = HikariConfigurationUtil.loadConfiguration(properties());

  return new HikariDataSource(config);
}

@Bean
public SpringLiquibase liquibase(DataSource dataSource) {
  SpringLiquibase liquibase = new SpringLiquibase();
  liquibase.setDataSource(dataSource);
  liquibase.setChangeLog("classpath:dbChangelog.xml");
  liquibase.setContexts("test");
  liquibase.setDropFirst(true);
  liquibase.setShouldRun(true);

  return liquibase;
}
{% endhighlight %}

The Spring enabled `Application` class should now look something like this:

file: `src/main/java/com/tutorial/Application.java`:

{% highlight java %}
@Configuration
@ComponentScan
@EnableTransactionManagement
public class Application {

  public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(Application.class);
    PersonRepository repository = ctx.getBean(PersonRepository.class);

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

    // close application context
    ((AnnotationConfigApplicationContext) ctx).close();
  }

  private Properties properties() {
    Properties props = new Properties();
    props.setProperty("javax.persistence.provider", "org.hibernate.jpa.HibernatePersistenceProvider");
    props.setProperty("javax.persistence.schema-generation.database.action", "none");
    props.setProperty("hibernate.hikari.dataSourceClassName", "org.h2.jdbcx.JdbcDataSource");
    props.setProperty("hibernate.hikari.dataSource.url", "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1;MVCC=true");
    props.setProperty("hibernate.hikari.dataSource.user", "sa");
    props.setProperty("hibernate.hikari.dataSource.password", "");
    props.setProperty("hibernate.hikari.minimumIdle", "5");
    props.setProperty("hibernate.hikari.maximumPoolSize", "10");
    props.setProperty("hibernate.hikari.idleTimeout", "30000");
    props.setProperty("hibernate.connection.handling_mode", "delayed_acquisition_and_hold");
    props.setProperty("hibernate.connection.provider_class", "org.hibernate.hikaricp.internal.HikariCPConnectionProvider");

    return props;
  }

  @Bean(destroyMethod = "close")
  public DataSource dataSource() {
    HikariConfig config = HikariConfigurationUtil.loadConfiguration(properties());

    return new HikariDataSource(config);
  }

  @Bean
  public SpringLiquibase liquibase(DataSource dataSource) {
    SpringLiquibase liquibase = new SpringLiquibase();
    liquibase.setDataSource(dataSource);
    liquibase.setChangeLog("classpath:dbChangelog.xml");
    liquibase.setContexts("test");
    liquibase.setDropFirst(true);
    liquibase.setShouldRun(true);

    return liquibase;
  }

  @Bean
  public LocalContainerEntityManagerFactoryBean entityManager() {
    LocalContainerEntityManagerFactoryBean entityManager = new LocalContainerEntityManagerFactoryBean();
    entityManager.setPackagesToScan("com.tutorial.entity");
    entityManager.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
    entityManager.setJpaProperties(properties());
    entityManager.setPersistenceUnitName("com.juliuskrah.tutorial");

    return entityManager;
  }

  @Bean
  public PlatformTransactionManager transactionManager(EntityManagerFactory emf) {

    return new JpaTransactionManager(emf);
  }
}
{% endhighlight %}

# Conclusion
In this post we converted the previous example to a Spring project. We also learned how to switch from `Application Managed`
(`private EntityManagerFactory emf = Persistence.createEntityManagerFactory("com.juliuskrah.tutorial")`) Entity 
Manager to `Container Managed` (`@PersistenceUnit(unitName = "com.juliuskrah.tutorial") private EntityManagerFactory emf`) Entity
Manager.    
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.

[JDK]:          http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:        http://maven.apache.org
[Spring]:       https://spring.io/docs/reference
