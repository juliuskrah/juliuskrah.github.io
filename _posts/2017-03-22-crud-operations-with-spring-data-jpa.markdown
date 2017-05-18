---
layout:     series
title:      CRUD Operations with Spring Data JPA
date:       2017-03-22 18:00:47 +0000
categories: tutorial
tags:       java maven hibernate jpa spring-data spring
section:    series
author:     juliuskrah
repo:       java-crud/tree/spring-data-hibernate-jpa
---
> [Spring Data JPA][Spring Data]{:target="_blank"} provides a familiar and consistent, Spring-based programming model for data access
  while still retaining the special traits of the underlying data store.  
  It makes it easy to use data access technologies, relational and non-relational databases, map-reduce frameworks, and cloud-based 
  data services. 

# Introduction
This is the fifth part part of a series of posts focused on `Hibernate` and `JPA`. In my previous post we saw how to abstract away
most of the logic written in prior post with [Spring]({% post_url 2017-03-05-crud-operations-with-spring %}). In this post we will
look at abstracting away the CRUD layer to [Spring Data JPA][Spring Data]{:target="_blank"} and writing less boilerplate code.   
We will end this post by externalizing our database configuration to a `.properties` file.

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
|  |  |__resources/
|  |  |  |__application.properties
|  |  |  |__dbChangelog.xml
|  |  |  |__log4j2.properties
|__pom.xml
```

# Setting up Dependencies
To enable the project to use Spring Data we need the following dependencies

- spring-data-jpa

Modify the `pom.xml` file by adding the following dependencies:

{% highlight xml %}
<dependencies>
  ...
  <dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-jpa</artifactId>
    <version>1.11.1.RELEASE</version>
  </dependency>
</dependencies>
{% endhighlight %}

# Refactoring the PersonRepository Interface
We would extend the Spring Data `Repository` marker interface. The general purpose here is to hold type information as well as being
able to discover interfaces that extend this one during classpath scanning for easy Spring bean creation.

file: `src/main/java/com/tutorial/repository/PersonRepository.java`:

{% highlight java %}
public interface PersonRepository extends Repository<Person, Long> {...}
{% endhighlight %}

We specify the generic type arguments as the `Person` entity and id type (`Long`).  
Next we will add a Spring Data repository scanning mechanism (`@EnableJpaRepositories`) to our configuration class to detect this
interface.

file: `src/main/java/com/tutorial/Application.java`:

{% highlight java %}
@Configuration
@EnableJpaRepositories
public class Application {...}
{% endhighlight %}

Spring Data JPA requires a bean named `entityManagerFactory`; we will modify the `LocalContainerEntityManagerFactoryBean` as follows:

file: `src/main/java/com/tutorial/Application.java`:

{% highlight java %}
@Bean(name = "entityManagerFactory")
public LocalContainerEntityManagerFactoryBean entityManager() {...}
{% endhighlight %}

Spring Data uses several query lookup strategies to create queries based on method names. 
You can read more on the [reference documentation][Spring Data Reference]{:target="_blank"}.
To support this so as not to write too much code we will refactor the `PersonRepository`.

file: `src/main/java/com/tutorial/repository/PersonRepository.java`:

{% highlight java %}
public interface PersonRepository extends Repository<Person, Long> {
    // Save or Update
    Optional<Person> save(Person person);

    Optional<Person> findOne(Long id);

    void delete(Person person);
}
{% endhighlight %}

Spring Data Repositories are transactional by default that's why the above snippet is missing the `@Transactional` annotation.

# Externalizing Configuration
Up until now we have been hardcoding configuration values into our source code. In a big project you would often find yourself 
writing configuration values in external files and programmatically calling those values in your source code.  
Create a file `application.properties`.

file: `src/main/resources/application.properties`:

{% highlight properties %}
spring.jpa.hibernate.ddl-auto=none
spring.jpa.properties.javax.persistence.provider=org.hibernate.jpa.HibernatePersistenceProvider
spring.jpa.properties.hibernate.hikari.dataSourceClassName=org.h2.jdbcx.JdbcDataSource
spring.jpa.properties.hibernate.hikari.dataSource.url=jdbc:h2:mem:test;DB_CLOSE_DELAY=-1;MVCC=true
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

The `application.properties` file will be loaded into our application using Spring's [`Resource`][Spring Resource]{:target="_blank"}
lookup mechanism.

file: `src/main/java/com/tutorial/Application.java`:

{% highlight java %}
@Configuration
@EnableJpaRepositories
@PropertySource("classpath:application.properties")
public class Application {
    @Inject
    private Environment env;
    ...
}
{% endhighlight %}

A Spring `Environment` in injected to read property values by key. The full `Application` class is provided below.

file: `src/main/java/com/tutorial/Application.java`:

{% highlight java %}
@Configuration
@EnableJpaRepositories
@EnableTransactionManagement
@PropertySource("classpath:application.properties")
public class Application {

    @Inject
    private Environment env;
    @Inject
    private ResourceLoader resourceLoader;

    public static void main(String[] args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(Application.class);
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
            System.out.format("Person from database: %s", consumer);
        });
        // Delete person
        repository.delete(p.get());

        p = Optional.empty();

        p = repository.findOne(1L);

        ((AnnotationConfigApplicationContext) ctx).close();
    }

    private Properties properties() {
        Properties props = new Properties();
        props.setProperty("javax.persistence.provider", env.getRequiredProperty("spring.jpa.properties.javax.persistence.provider"));
        props.setProperty("javax.persistence.schema-generation.database.action", env.getRequiredProperty("spring.jpa.hibernate.ddl-auto"));
        props.setProperty("hibernate.hikari.dataSourceClassName", env.getRequiredProperty("spring.jpa.properties.hibernate.hikari.dataSourceClassName"));
        props.setProperty("hibernate.hikari.dataSource.url", env.getRequiredProperty("spring.jpa.properties.hibernate.hikari.dataSource.url"));
        props.setProperty("hibernate.hikari.dataSource.user", env.getRequiredProperty("spring.jpa.properties.hibernate.hikari.dataSource.user"));
        props.setProperty("hibernate.hikari.dataSource.password", env.getRequiredProperty("spring.jpa.properties.hibernate.hikari.dataSource.password"));
        props.setProperty("hibernate.hikari.minimumIdle", env.getRequiredProperty("spring.jpa.properties.hibernate.hikari.minimumIdle"));
        props.setProperty("hibernate.hikari.maximumPoolSize", env.getRequiredProperty("spring.jpa.properties.hibernate.hikari.maximumPoolSize"));
        props.setProperty("hibernate.hikari.idleTimeout", env.getRequiredProperty("spring.jpa.properties.hibernate.hikari.idleTimeout"));
        props.setProperty("hibernate.connection.handling_mode", env.getRequiredProperty("spring.jpa.properties.hibernate.connection.handling_mode"));
        props.setProperty("hibernate.connection.provider_class", env.getRequiredProperty("spring.jpa.properties.hibernate.connection.provider_class"));

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
        liquibase.setChangeLog(env.getRequiredProperty("liquibase.change-log"));
        liquibase.setContexts(env.getRequiredProperty("liquibase.contexts"));
        liquibase.setDropFirst(env.getRequiredProperty("liquibase.drop-first", Boolean.class));
        liquibase.setShouldRun(env.getRequiredProperty("liquibase.enabled", Boolean.class));
		
        return liquibase;
    }

    @PostConstruct
    public void checkChangelogExists() {
        if (env.getRequiredProperty("liquibase.check-change-log-location", Boolean.class)) {
            Resource resource = this.resourceLoader.getResource(env.getRequiredProperty("liquibase.change-log"));
            Assert.state(resource.exists(), "Cannot find changelog location: " + resource
                + " (please add changelog or check your Liquibase configuration)");
        }
    }

    @Bean(name = "entityManagerFactory")
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
In this post we added Spring Data JPA to reduce CRUD boilerplate code. We also extracted configuration values into an external file.  
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.


[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org
[Spring Data]:              http://projects.spring.io/spring-data/
[Spring Data Reference]:    http://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.query-methods.details
[Spring Resource]:          http://docs.spring.io/spring/docs/current/spring-framework-reference/html/resources.html
