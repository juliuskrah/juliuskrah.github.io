---
layout:     post
title:      Auditing with Hibernate Envers
date:       2018-03-15 22:15:20 +0000
categories: blog
tags:       java maven jpa spring-boot hibernate audit
section:    blog
author:     juliuskrah
repo:       spring-data-audit/tree/hibernate-envers
---
> [Envers][]{:target="_blank"} is a core Hibernate model that works both with Hibernate and JPA.

# Introduction

You can apply auditing to your JPA entities either by using [Spring Data]({%post_url 2018-02-15-auditing-with-spring-data-jpa %})
or `Hibernate Envers`. You can also combine both to provide a rich auditing infrastructure for your applications.

Hibernate Envers provides an easy auditing / versioning solution for your entity classes. Envers has the 
following core features:

- Auditing of all mappings defined by the JPA specification
- Auditing some Hibernate mappings that extend JPA, e.g. custom types and collections/maps of "simple" types
  like Strings, Integers.
- Logging data for each revision using a "revision entity"
- Querying historical snapshots of an entity and its associations

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
[zip](https://github.com/juliuskrah/spring-data-audit/archive/v1.2.zip)/[tar.gz](https://github.com/juliuskrah/spring-data-audit/archive/v1.2.tar.gz).
Go ahead download and extract, and import into your favorite IDE as a maven project. Run and confirm everything 
works.

# Create the Required Auditing Classes

The first order of business is to create an Entity class and add the Hibernate auditing metadata:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/audit/Customer.java' %}

{% highlight java %}
@Entity
@Audited
public class Customer implements Serializable {
  private static final long serialVersionUID = 1L;

  @Id
  @GeneratedValue(generator = "uuid2")
  @GenericGenerator(name = "uuid2", strategy = "uuid2")
  private String id;
  private String name;
  private String email;
  private String item;
  @CreationTimestamp
  @Column(updatable = false)
  private LocalDateTime createdDate;
  @UpdateTimestamp
  private LocalDateTime lastModifiedDate;

  // Getters and Setters omitted for brevity
}
{% endhighlight %}

> `@Audited` is the primary Hibernate annotation for JPA auditing.

`@Audited` when applied to an Entity class audits all persistent properties. You can also annotate only some
persistent properties with `@Audited`. This will cause only these properties to be audited.

The audit (history) of an entity can be accessed using the `AuditReader` interface, which can be obtained
having an open `EntityManager` via the `AuditReaderFactory`.

e.g.

```java
AuditReader auditReader = AuditReaderFactory.get(entityManager);

List<Number> revisions = auditReader.getRevisions(ApplicationCustomer.class, 1L);

CustomTrackingRevisionEntity revEntity = auditReader.findRevision(
    CustomTrackingRevisionEntity.class,
    revisions.get( 0 )
);
```

Now, considering the previous `Customer` entity, let’s see how Envers auditing works when inserting, updating,
and deleting the entity in question:

file: {% include file-path.html file_path='src/test/java/com/juliuskrah/audit/ApplicationTests.java' %}

{% highlight java %}
@SpringBootTest
@SpringJUnitConfig
public class ApplicationTests {

  @PersistenceUnit
  private EntityManagerFactory emf;
  private Customer customer;

  private EntityManagerFactory entityManagerFactory() {
    return this.emf;
  }

  @BeforeEach
  public void init() {
    this.customer = new Customer();
    this.customer.setName("Julius");
    this.customer.setEmail("juliuskrah@example.com");
  }

  @Test
  public void testAudits() {
    // Insert
    doInJPA(this::entityManagerFactory, entityManager -> {
      entityManager.persist(this.customer);
      assertTrue(entityManager.contains(this.customer));
    });

    String id = this.customer.getId();

    Customer firstRevision = doInJPA(this::entityManagerFactory, entityManager -> {
      return AuditReaderFactory.get(entityManager).find(Customer.class, id, 1);
    });

    assertThat(firstRevision).isNotNull();
    assertThat(firstRevision.getCreatedDate()).isNotNull();
    assertThat(firstRevision.getCreatedDate()).isBefore(now());
    assertThat(firstRevision.getName()).isSameAs("Julius");

    // Retrieve and Update
    doInJPA(this::entityManagerFactory, entityManager -> {
      Customer customer = entityManager.find(Customer.class, id);
      customer.setItem("Biscuits");
      customer.setName("Abeiku");
    });

    Customer secondRevision = doInJPA(this::entityManagerFactory, entityManager -> {
      return AuditReaderFactory.get(entityManager).find(Customer.class, id, 2);
    });

    assertThat(secondRevision).isNotNull();
    assertThat(secondRevision.getLastModifiedDate()).isNotNull();
    assertThat(secondRevision.getLastModifiedDate()).isAfter(firstRevision.getCreatedDate());
    assertThat(secondRevision.getName()).isSameAs("Abeiku");

    // Delete
    doInJPA(this::entityManagerFactory, entityManager -> {
      Customer customer = entityManager.getReference(Customer.class, id);
      entityManager.remove(customer);
    });

    Customer thirdRevision = doInJPA(this::entityManagerFactory, entityManager -> {
      return AuditReaderFactory.get(entityManager).find(
        Customer.class,
        Customer.class.getName(),
        id,
        3,
        true);
    });

    assertThat(thirdRevision).isNotNull();
    assertThat(thirdRevision.getLastModifiedDate()).isNull();
    assertThat(thirdRevision.getName()).isNull();
  }
}
{% endhighlight %}

Unlike Spring Auditing, Hibernate Envers writes audit infomation in a different table, so delete infomation
is preserved. It writes into `entity_table_AUD` (`customer_AUD` in our case) by default if not explicitely
configured.

That's all folks

# Conclusion

In this post we looked at Hibernate Envers auditing infrastructure with minimal configuration leveraing auto
configuration from Spring-Boot.

You can find the source to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.

[Envers]:                   http://hibernate.org/orm/envers/
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org
[Spring Data]:              https://docs.spring.io/spring-data/commons/docs/current/reference/html/