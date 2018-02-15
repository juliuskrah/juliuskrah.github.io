---
layout:     post
title:      Auditing with Spring Data JPA
date:       2018-02-15 10:00:20 +0000
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
|  |  |  |  |  |  |__Customer.java
|  |  |  |  |  |  |__CustomerHandler.java
|  |  |  |  |  |  |__CustomerRepository.java
|  |  |  |  |  |  |__SpringSecurityAuditorAware .java
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
    return ReactiveSecurityContextHolder.getContext()
      .map(SecurityContext::getAuthentication)
      .filter(Authentication::isAuthenticated)
      .map(Authentication::getName)
      .switchIfEmpty(Mono.just("julius"))
      .blockOptional();
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

> Take note of the `auditorAwareRef` attribute with value `springSecurityAuditorAware`. The 
  `springSecurityAuditorAware` is the bean name of the `AuditorAware` implementation.

## Test the `@CreatedDate` and `@LastModifiedDate`
Now we need to verify the auditing infrastructure works as expected. We will write two test cases to verify this:

file: {% include file-path.html file_path='src/test/java/com/juliuskrah/audit/ApplicationTests.java' %}

{% highlight java %}
@RunWith(SpringRunner.class)
@SpringBootTest
public class ApplicationTests {
  @Autowired
  private CustomerRepository repository;

  @Test
  public void customerWhenCreateThenCreatedDateIsNotNull() {
    Customer customer = new Customer();
    customer.setName("James Gunn");
    customer.setEmail("jamesgunn@gmail.com");

    repository.save(customer);

    assertThat(customer.getId()).isNotNull();
    assertThat(customer.getCreatedDate()).isNotNull();   // CreatedBy added by Spring Audit
    assertThat(customer.getCreatedDate()).isBefore(now());
  }
	
  @Test
  public void customerWhenUpdateThenLastModifiedDateIsAfterCreatedDate() {
    Customer customer = new Customer();
    customer.setName("Tyler Perry");
    customer.setEmail("tylerperry@gmail.com");

    repository.save(customer);
    Optional<Customer> c = repository.findById(customer.getId());
		
    assertThat(c.isPresent()).isTrue();
		
    customer = c.get();
    customer.setOrder("Samsung Galaxy");
		
    customer = repository.save(customer);
		
    assertThat(customer.getLastModifiedDate()).isAfter(customer.getCreatedDate()); // Last Modified By added by Spring Audit
  }
}
{% endhighlight %}

# Add Some Routes
This step is added to test the Principal injected for `@CreatedBy` and `@LastModifiedBy`. We will first create
a `HandlerFuntion` to process our requests:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/audit/CustomerHandler.java' %}

{% highlight java %}
@Component
public class CustomerHandler {
  private final CustomerRepository repository;

  public CustomerHandler(CustomerRepository repository) {
    this.repository = repository;
  }

  public Mono<ServerResponse> retrieveCustomers(ServerRequest request) {
    Iterable<Customer> customers = repository.findAll();
    return ServerResponse.ok().body(Mono.just(customers), 
      new ParameterizedTypeReference<Iterable<Customer>>() {});
  }

  public Mono<ServerResponse> retrieveCustomer(ServerRequest request) {
    String id = request.pathVariable("id");
    Optional<Customer> customer = repository.findById(id);
    if (customer.isPresent())
      return ServerResponse.ok().body(Mono.just(customer.get()), Customer.class);
    return ServerResponse.notFound().build();
  }

  public Mono<ServerResponse> createCustomer(ServerRequest request) {
    Mono<Customer> customer = request.bodyToMono(Customer.class);

    return customer.flatMap(c -> {
      repository.save(c);
      URI uri = request.uriBuilder().path("/{id}").build(c.getId());
      return ServerResponse.created(uri).build();
    });
  }

  public Mono<ServerResponse> updateCustomer(ServerRequest request) {
    Mono<Customer> customer = request.bodyToMono(Customer.class);

    return customer.flatMap(c -> {
      String id = request.pathVariable("id");
      Optional<Customer> custom = repository.findById(id);
      if (custom.isPresent()) {
        String order = c.getOrder();
        c = custom.get();
        c.setOrder(order);
        repository.save(c);
      } else
        return ServerResponse.notFound().build();
      return ServerResponse.noContent().build();
    });
  }

  public Mono<ServerResponse> deleteCustomer(ServerRequest request) {
    String id = request.pathVariable("id");
    repository.deleteById(id);

    return ServerResponse.noContent().build();
  }
}
{% endhighlight %}

Add a `RouterFunction` to the main configuration class:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/audit/Application.java' %}

{% highlight java %}
@EnableJpaAuditing
@SpringBootApplication
public class Application {
  //
    
  @Bean
  public RouterFunction<ServerResponse> customerRouter(CustomerHandler customerHandler) {
    return nest(path("/customers"),
      route(GET("/").and(accept(APPLICATION_JSON)), customerHandler::retrieveCustomers)
      .andRoute(GET("/{id}").and(accept(APPLICATION_JSON)), customerHandler::retrieveCustomer)
      .andRoute(POST("/").and(contentType(APPLICATION_JSON)), customerHandler::createCustomer)
      .andRoute(PUT("/{id}").and(contentType(APPLICATION_JSON)), customerHandler::updateCustomer)
      .andRoute(DELETE("/{id}"), customerHandler::deleteCustomer));
  }
}
{% endhighlight %}

Create a default username and password:

file: {% include file-path.html file_path='src/main/resources/application.yaml' %}

{% highlight yaml %}
spring:
  security:
    user:
      name: julius
      password: secret
{% endhighlight %}

## Test the `@CreatedBy` and `@LastModifiedBy`
We need to verify the auditing infrastructure works as expected. We will write two test cases to verify this:

file: {% include file-path.html file_path='src/test/java/com/juliuskrah/audit/ApplicationTests.java' %}

{% highlight java %}
@SpringBootTest
public class ApplicationTests {
  //

  @Autowired
  private ApplicationContext context;
  private WebTestClient rest;

  @Before
  public void setup() {
    this.rest = WebTestClient
      .bindToApplicationContext(this.context)
      .apply(springSecurity())
      .configureClient()
      .filter(basicAuthentication("julius", "secret"))
      .build();
  }

  @Test
  public void customerWhenCreateThenCreatedByIsNotNull() {
    Customer customer = new Customer();
    customer.setName("Jack Sparrow");
    customer.setEmail("jacksparrow@strangertides.com");
		
    this.rest
      .mutateWith(csrf())
      .post()
      .uri("/customers")
      .body(Mono.just(customer), Customer.class)
      .exchange()
      .expectStatus().isCreated();
		
    this.rest
      .get()
      .uri("/customers")
      .exchange()
      .expectStatus().isOk()
      .expectBody()
      .jsonPath("$[0].createdBy").exists();
  }
	
  @Test
  public void customerWhenUpdateThenLastModifiedDateIsNotNull() {
    Customer customer = new Customer();
    customer.setName("Elizabeth Swarn");
    customer.setEmail("lizzyswarn@strangertides.com");
		
    repository.save(customer);
    customer.setOrder("food stuff");
		
    this.rest
      .mutateWith(csrf())
      .put()
      .uri("/customers/{id}", customer.getId())
      .body(Mono.just(customer), Customer.class)
      .exchange()
      .expectStatus().isNoContent();
		
    this.rest
      .get()
      .uri("/customers/{id}", customer.getId())
      .exchange()
      .expectStatus().isOk()
      .expectBody()
      .jsonPath("$.lastModifiedBy").exists();
  }
}
{% endhighlight %}


That's all folks

# Conclusion
In this post we looked at Spring Data's built-in auditing infrastructure with minimal configuration leveraging
Spring Webflux and Reactive Spring Security.   
You can find the source to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.

[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org
[Spring Data]:              https://docs.spring.io/spring-data/commons/docs/current/reference/html/
