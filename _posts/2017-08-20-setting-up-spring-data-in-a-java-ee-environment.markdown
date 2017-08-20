---
layout:     post
title:      Setting up Spring Data in a Java EE environment
date:       2017-08-20 19:47:56 +0000
categories: blog
tags:       java maven javaee spring-data jax-rs jpa
section:    blog
author:     juliuskrah
repo:       javaee-spring-data/tree/master
---
> The [Java EE platform][]{:target="_blank"} is developed through the Java Community Process ([JCP][]{:target="_blank"}),
  which is responsible for all Java technologies. Expert groups composed of interested parties have created
  Java Specification Requests (JSRs) to define the various Java EE technologies. The work of the Java Community under the
  JCP program helps to ensure Java technology's standards of stability and cross-platform compatibility.

# Introduction
In this post we are going to learn how to use [Spring Data]({% post_url 2017-03-22-crud-operations-with-spring-data-jpa %})
in a Java EE environment using JAX-RS.

## Project Structure
At the end of this guide our folder structure will look similar to the following:

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__juliuskrah
|  |  |  |  |  |__ApplicationConfig.java
|  |  |  |  |  |__LocalDateAdapter.java
|  |  |  |  |  |__LocalDateTimeAdapter.java
|  |  |  |  |  |__package-info.java
|  |  |  |  |  |__Person.java
|  |  |  |  |  |__PersonRepository.java
|  |  |  |  |  |__PersonService.java
|  |  |__resources/
|  |  |  |__META-INF/
|  |  |  |  |__persistence.xml
|__pom.xml
```

You can also find the complete sample {% include source.html %}.

# Prerequisites
To follow along this guide, your development system should have the following setup:
- [Java Development Kit][JDK]{:target="_blank"}  
- [Maven][]{:target="_blank"}
- [cURL][]{:target="_blank"}

[Generate]({% post_url 2017-07-16-developing-restful-services-with-jax-rs-jersey %}#creating-project-template)
a maven project template and let us begin.

## Project Dependencies
To get started, we need to gather our project dependencies. This is a maven based project, and we can easily declare
our dependencies in a `pom.xml` file.

file: {% include file-path.html file_path='pom.xml' %}

{% highlight xml %}
...
<properties>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <maven.compiler.source>1.8</maven.compiler.source>
  <maven.compiler.target>1.8</maven.compiler.target>

  <h2.version>1.4.196</h2.version>
  <hibernate.version>5.2.10.Final</hibernate.version>
  <javaee-api.version>7.0</javaee-api.version>
  <spring-data-releasetrain.version>Ingalls-SR6</spring-data-releasetrain.version>
  <wildfly-swarm.version>2017.8.1</wildfly-swarm.version>
</properties>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.data</groupId>
      <artifactId>spring-data-releasetrain</artifactId>
      <version>${spring-data-releasetrain.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>${hibernate.version}</version>
    <scope>provided</scope>
  </dependency>

  <!-- Java EE 7 dependency -->
  <dependency>
    <groupId>javax</groupId>
    <artifactId>javaee-api</artifactId>
    <version>${javaee-api.version}</version>
    <scope>provided</scope>
  </dependency>

  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>${h2.version}</version>
  </dependency>
</dependencies>

<build>
  <finalName>${project.artifactId}</finalName>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-war-plugin</artifactId>
      <version>3.1.0</version>
      <configuration>
        <failOnMissingWebXml>false</failOnMissingWebXml>
      </configuration>
    </plugin>
    <plugin>
      <groupId>org.wildfly.swarm</groupId>
      <artifactId>wildfly-swarm-plugin</artifactId>
      <version>${wildfly-swarm.version}</version>
      <configuration>
        <properties>
          <swarm.context.path>${project.build.finalName}</swarm.context.path>
        </properties>
      </configuration>
      <executions>
        <execution>
          <goals>
            <goal>package</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
{% endhighlight %}

We have declared dependencies on `Spring-Data`, `Hibernate` and `Java EE 7`. `H2` embedded database has also been added
for `JPA` access. In the plugins section we have `wildfly-Swarm` which will take care of setting up a Java EE environment for this
project.

## JPA Setup
Spring Data JPA as the name implies uses [JPA]({% post_url 2017-02-15-crud-operations-with-hibernate-and-jpa %}), and for this reason we are going to setup JPA:

file: {% include file-path.html file_path='src/main/resources/META-INF/persistence.xml' %}

{% highlight xml %}
<persistence version="2.1"
	xmlns="http://xmlns.jcp.org/xml/ns/persistence" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence
 		http://xmlns.jcp.org/xml/ns/persistence/persistence_2_1.xsd">
  <persistence-unit name="javaee-spring-dataPU" transaction-type="JTA">
    <properties>
      <property name="javax.persistence.schema-generation.database.action" value="drop-and-create"/>
      <property name="javax.persistence.schema-generation.create-source" value="metadata"/>
      <property name="javax.persistence.schema-generation.drop-source" value="metadata"/>
      <property name="hibernate.show_sql" value="true"/>
    </properties>
  </persistence-unit>
</persistence>
{% endhighlight %}

Next is to create an Entity class to be managed by the Entity Manager of the Java EE Container:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/Person.java' %}

{% highlight java %}
@Entity
public class Person implements Serializable {
  private static final long serialVersionUID = 1L;

  @Id
  @GeneratedValue(generator = "uuid2")
  @GenericGenerator(name = "uuid2", strategy = "uuid2")
  private String id;

  private String firstName;

  private String lastName;

  private LocalDate dateOfBirth;

  private LocalDateTime createdDate;

  private LocalDateTime modifiedDate;

  // Getters and Setters omitted for brevity
}
{% endhighlight %}

## Spring Data JPA and JAX-RS
With JPA setup (albeit simply), we can now start using Spring Data:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/PersonRepository.java' %}

{% highlight java %}
public interface PersonRepository extends JpaRepository<Person, String> {
  // Nothing here
}
{% endhighlight %}

Create the resource class to respond to HTTP requests over [REST]({% post_url 2017-07-16-developing-restful-services-with-jax-rs-jersey %})
and access Spring Data Repository abstraction:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/PersonService.java' %}

{% highlight java %}
// The @Stateless annotation eliminates the need for manual transaction demarcation
@Stateless
@Path("/v1.0/persons")
public class PersonService {
  private static final Logger LOGGER = Logger.getLogger(PersonService.class.getName());
  @PersistenceContext
  private EntityManager entityManager;
  private PersonRepository personRepository;

  @PostConstruct
  private void init() {
    // Instantiate Spring Data factory
    RepositoryFactorySupport factory = new JpaRepositoryFactory(entityManager);
    // Get an implemetation of PersonRepository from factory
    this.personRepository = factory.getRepository(PersonRepository.class);
  }

  /**
   * POST /api/v1.0/persons : Create a new person.
   *
   * @param person the person to create
   * @return the Response with status 201 (Created) and with location of newly created resource
   * or with status 400 (Bad Request) if the person has
   * already an ID
   */
  @POST
  public Response createPerson(Person person) {
    LOGGER.log(Level.INFO, "REST request to create Person : {0}", person);
    if (Objects.nonNull(person.getId()))
      throw new WebApplicationException(Response.Status.BAD_REQUEST);
    person.setCreatedDate(LocalDateTime.now());
    this.personRepository.save(person);

    return Response.created(URI.create("/v1.0/persons/" + person.getId())).build();
  }

  /**
   * GET /api/v1.0/persons : get all the people.
   *
   * @return the list of people with status 200 (OK)
   */
  @GET
  public List<Person> findPeople() {
    LOGGER.log(Level.INFO, "REST request to fetch all Persons");
    return this.personRepository.findAll();
  }


  /**
   * GET /api/v1.0/persons/:id : get the "id" person.
   *
   * @param id the id of the person to retrieve
   * @return the Response with status 200 (OK) and with body the person,
   * or with status 404 (Not Found)
   */
  @GET
  @Path("{id: [0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}}")
  public Response findPerson(@PathParam("id") String id) {
    LOGGER.log(Level.INFO, "REST request to fetch Person : {0}", id);
    Person person = this.personRepository.findOne(id);
    if (Objects.isNull(person))
      throw new WebApplicationException(Response.Status.NOT_FOUND);
    return Response.ok(person).build();
  }

  /**
   * PUT /api/v1.0/persons/:id : Updates an existing person.
   *
   * @param id     the id of the person to retrieve
   * @param person the person to update
   * @return the Response with status 204 (NO CONTENT) and with no body,
   * or with status 400 (Bad Request) if the person is not
   * valid, or with status 500 (Internal Server Error) if the person
   * couldn't be updated
   */
  @PUT
  @Path("{id: [0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}}")
  public Response updatePerson(@PathParam("id") String id, Person person) {
    LOGGER.log(Level.INFO, "REST request to update Person : {0}", person);
    Person person1 = this.personRepository.findOne(id);
    if (Objects.isNull(person1))
      throw new WebApplicationException(Response.Status.NOT_FOUND);
    person.setId(id);
    person.setCreatedDate(person1.getCreatedDate());
    person.setModifiedDate(LocalDateTime.now());
    this.personRepository.save(person);
    return Response.noContent().build();
  }

  /**
   * DELETE  /api/v1.0/persons/:id : delete the "id" person.
   *
   * @param id the id of the person to delete
   * @return the Response with status 204 (NO CONTENT)
   */
  @DELETE
  @Path("{id: [0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}}")
  public Response deletePerson(@PathParam("id") String id) {
    LOGGER.log(Level.INFO, "REST request to delete Person : {0}", id);
    this.personRepository.delete(id);
    return Response.noContent().build();
  }
}
{% endhighlight %}

Finally let us activate JAX-RS with:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/ApplicationConfig.java' %}

{% highlight java %}
/**
 * A class extending {@link Application} and annotated with @ApplicationPath is the Java EE 7 "no XML" approach to activating
 * JAX-RS.
 * <p>
 * <p>
 * Resources are served relative to the servlet path specified in the {@link ApplicationPath} annotation.
 * </p>
 */
@ApplicationPath("api")
public class ApplicationConfig extends Application {
    /* class body intentionally left blank */
}
{% endhighlight %}

We are using Java 8 data-types of `LocalDateTime` and `LocalDate` and the `Jackson` JSON serializer/deserializer in use needs to know how
marshal/unmarshal these types. One way to do this is add the `jackson-datatype-jsr310` jar to the classpath and register the module
with the `ObjectMapper`.  
In our situation we only need to serialize and deserialize two data-types which we can wire manually instead of adding another dependency:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/LocalDateTimeAdapter.java' %}

{% highlight java %}
public class LocalDateTimeAdapter extends XmlAdapter<String, LocalDateTime> {

  @Override
  public LocalDateTime unmarshal(String dateInput) {
    return LocalDateTime.parse(dateInput, DateTimeFormatter.ISO_DATE_TIME);
  }

  @Override
  public String marshal(LocalDateTime localDateTime) {
    return DateTimeFormatter.ISO_DATE_TIME.format(localDateTime);
  }
}
{% endhighlight %}

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/LocalDateAdapter.java' %}

{% highlight java %}
public class LocalDateAdapter extends XmlAdapter<String, LocalDate> {

  @Override
  public LocalDate unmarshal(String dateInput) {
    return LocalDate.parse(dateInput, DateTimeFormatter.ISO_DATE);
  }

  @Override
  public String marshal(LocalDate localDate) {
    return DateTimeFormatter.ISO_DATE.format(localDate);
  }
}
{% endhighlight %}

Then we register it to be used for all `LocalDate` and `LocalDateTime` instances in the application.

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/package-info.java' %}

{% highlight java %}
@XmlJavaTypeAdapters({
      @XmlJavaTypeAdapter(type = LocalDateTime.class,
            value = LocalDateTimeAdapter.class),
      @XmlJavaTypeAdapter(type = LocalDate.class,
            value = LocalDateAdapter.class)
})
package com.juliuskrah;
{% endhighlight %}

## Testing the Application
Package the application with wildfly-swarm by running:

{% highlight bash %}
> mvn clean package
{% endhighlight %}

This will create a jar file: `${project.build.finalName}-swarm.jar` in the `target` directory.  
Run the following command to start an `Undertow HTTP Server` on port `8080` and on context path
`/${project.artifactId}` (in my case `/javaee-spring-data`).

{% highlight bash %}
> java -jar target\javaee-spring-data-swarm.jar # Windows
> java -jar target/*.jar                        # Linux and Mac
{% endhighlight %}

Open another shell window and test the application:

{% highlight bash %}
> curl -i -X POST -H "Content-Type: application/json" -d "{\"firstName\": \"John\",\"lastName\": \"Doe\", \"dateOfBirth\":  \"2017-08-19\"}" http://localhost:8080/javaee-spring-data/api/v1.0/persons

HTTP/1.1 201 Created
Connection: keep-alive
Location: http://localhost:8080/javaee-spring-data/api/v1.0/persons/ebd64f47-73ec-4829-a79d-32b8c9c83186
Content-Length: 0
Date: Sun, 20 Aug 2017 19:14:48 GMT
{% endhighlight %}

Access the newly created person:

{% highlight bash %}
> curl -i -X GET -H "Accept: application/json" http://localhost:8080/javaee-spring-data/api/v1.0/persons/ebd64f47-73ec-4829-a79d-32b8c9c83186

HTTP/1.1 200 OK
Connection: keep-alive
Content-Type: application/json
Content-Length: 168
Date: Sun, 20 Aug 2017 19:21:57 GMT

{
  "id":"ebd64f47-73ec-4829-a79d-32b8c9c83186",
  "firstName": "John",
  "lastName": "Doe",
  "dateOfBirth": "2017-08-19",
  "createdDate": "2017-08-20T19:14:48.581",
  "modifiedDate": null
}
{% endhighlight %}

Update the person:

{% highlight bash %}
> curl -i -X PUT -H "Content-Type: application/json" -d "{\"firstName\": \"Jane\",\"lastName\": \"Doe\", \"dateOfBirth\":  \"2017-08-19\"}" http://localhost:8080/javaee-spring-data/api/v1.0/persons/ebd64f47-73ec-4829-a79d-32b8c9c83186

HTTP/1.1 204 No Content
Date: Sun, 20 Aug 2017 19:31:55 GMT
{% endhighlight %}

Delete the person:

{% highlight bash %}
> curl -i -X DELETE http://localhost:8080/javaee-spring-data/api/v1.0/persons/ebd64f47-73ec-4829-a79d-32b8c9c83186

HTTP/1.1 204 No Content
Date: Sun, 20 Aug 2017 19:34:59 GMT
{% endhighlight %}

# Conclusion
In this post we learned how to integrate Spring Data in a Java EE application using wildfly-Swarm as EE container.

You can find the source to this guide {% include source.html %}. Until the next post, keep doing cool things :smile:.

[cURL]:                              https://curl.haxx.se/
[Java EE Platform]:                  https://docs.oracle.com/javaee/7/tutorial/index.html
[JCP]:                               https://jcp.org/en/home/index
[JDK]:                               http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                             http://maven.apache.org
