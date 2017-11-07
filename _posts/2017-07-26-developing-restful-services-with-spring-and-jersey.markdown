---
layout:     series
title:      Developing RESTful Web Services with Spring and Jersey
date:       2017-07-26 21:39:54 +0000
categories: tutorial
tags:       java maven jersey jax-rs rest spring-boot javaee
section:    series
author:     juliuskrah
repo:       rest-example/tree/jersey-spring
---
> [Jersey framework][Jersey]{:target="_blank"} is a [JAX-RS][]{:target="_blank"} Reference Implementation. `Jersey`
  provides it’s own API that extend the JAX-RS toolkit with additional features and utilities to further simplify 
  RESTful service and client development. Jersey also exposes numerous extension SPIs so that developers may extend 
  Jersey to best suit their needs.

# Introduction
This post is a third in a [series]({% post_url 2017-07-23-developing-restful-services-with-spring %}) on 
[REST]({% post_url 2017-07-16-developing-restful-services-with-jax-rs-jersey %}). In this post we will combine
the power of [`Spring`][Spring]{:target="_blank"}  and `JAX-RS`(Jersey) to build a REST API.  
We will setup Spring to forward all requests to the Jersey Servlet for processing.

## Project Structure
At the end of this guide our folder structure will look similar to the following:

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__juliuskrah
|  |  |  |  |  |__Application.java
|  |  |  |  |  |__Resource.java
|  |  |  |  |  |__ResourceService.java
|__pom.xml
```

# Prerequisites
To follow along this guide, your development system should have the following applications installed:

- [Java Development Kit][JDK]{:target="_blank"}  
- [Maven][]{:target="_blank"}
- [cURL][]{:target="_blank"}

# Creating Project Template
Head over to the [Spring Initializr][Initializr]{:target="_blank"} website to generate a Spring project template:

![spring.io](https://i.imgur.com/6XjOoqC.png)

{:.image-caption}
*Spring Initializr*

Select `Jersey (JAX-RS)` and `DevTools` as dependencies and generate the project. `DevTools` is a handy tool to have 
during development as it offers _live reload_ when code changes.

Download and extract the template and let's get to work :smile:.

## Building Resources
Before we proceed, run the generated project to ensure it works:

{% highlight bash %}
mvnw clean spring-boot:run   # Windows
./mvnw clean spring-boot:run # Linux, Mac
{% endhighlight %}

If everything goes well, `Tomcat` should be started on port `8080` and wait for `http` requests.

## Setting up dependencies
Remove the `spring-boot-starter-web` dependency from the `pom.xml`. Refer to issue 
[`3132`](https://github.com/spring-projects/spring-boot/issues/3132){:target="_blank"}
of `Spring-Boot` for more insight on this bug:

file: {% include file-path.html file_path='pom.xml' %}

{% highlight xml %}
<!--
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
-->
{% endhighlight %}

## Building Resources
We will create a `POJO` to represent our REST resource.

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/Resource.java' %}

{% highlight java %}
@XmlRootElement
public class Resource {
    private Long id;
    private String description;
    private LocalDateTime createdTime;
    private LocalDateTime modifiedTime;

    // Default constructor

    // All Args constructor
    
    // Getters and Setters omitted for brevity
}
{% endhighlight %}

The next thing is to wire up a `ResourceService` to expose some endpoints.

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/ResourceService.java' %}

{% highlight java %}
@Path("/api/v1.0/resources")
@Produces({MediaType.APPLICATION_JSON, MediaType.APPLICATION_XML})
public class ResourceService {

    private static List<Resource> resources = null;

    static {
        resources = new ArrayList<>();
        resources.add(new Resource(1L, "Resource One", LocalDateTime.now(), null));
        resources.add(new Resource(2L, "Resource Two", LocalDateTime.now(), null));
        resources.add(new Resource(3L, "Resource Three", LocalDateTime.now(), null));
        resources.add(new Resource(4L, "Resource Four", LocalDateTime.now(), null));
        resources.add(new Resource(5L, "Resource Five", LocalDateTime.now(), null));
        resources.add(new Resource(6L, "Resource Six", LocalDateTime.now(), null));
        resources.add(new Resource(7L, "Resource Seven", LocalDateTime.now(), null));
        resources.add(new Resource(8L, "Resource Eight", LocalDateTime.now(), null));
        resources.add(new Resource(9L, "Resource Nine", LocalDateTime.now(), null));
        resources.add(new Resource(10L, "Resource Ten", LocalDateTime.now(), null));
    }

   /**
    * GET  /api/v1.0/resources : get all resources.
    * 
    * @return the {@code List<Resource>} of resources with status code 200 (OK)
    */
    @GET
    public List<Resource> getResources() {
        return resources;
    }
    ...
}
{% endhighlight %}

> For a production ready application, you will normally connect to a database. For the purpose of this tutorial,
  we will use a `static` field to initialize our list.

In the class with the `main` method; also annotated with [`@SpringBootApplication`](https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-using-springbootapplication-annotation.html){:target="_blank"},
add the following `@Bean` of type `ResourceConfig` in which you register all the endpoints:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/Application.java' %}

{% highlight java %}
...
@Bean
public ResourceConfig jerseyConfig() {
    ResourceConfig config = new ResourceConfig();
    config.register(ResourceService.class);
    return config;
}
{% endhighlight  %}

One more thing to wire `Jersey` to `Spring`, annotate your `ResourceService` class with `@Component` to
make it a Spring managed bean.

{% highlight java %}
@Component
public class ResourceService { ... }
{% endhighlight %}

Start the server by running the following command:

{% highlight bash %}
mvnw clean spring-boot:run   # Windows
./mvnw clean spring-boot:run # Linux, Mac
{% endhighlight %}

With the Tomcat server up and running, open another shell window and execute the following `cURL` command:

{% highlight bash %}
> curl -i -H "Accept: application/json" http://localhost:8080/api/v1.0/resources

HTTP/1.1 200 OK
Content-Type: application/json
content-length: 991
connection: keep-alive

[
  {
    "id": 1,
    "description": "Resource One",
    "createdTime": "2017-07-13T22:36:28.384",
    "modifiedTime": null
  },
  {
    "id": 2,
    "description": "Resource Two",
    "createdTime": "2017-07-13T22:36:28.384",
    "modifiedTime": null
  },
  {
    "id": 3,
    "description": "Resource Three",
    "createdTime": "2017-07-13T22:36:28.384",
    "modifiedTime": null
  },
  {
    "id": 4,
    "description": "Resource Four",
    "createdTime": "2017-07-13T22:36:28.384",
    "modifiedTime": null
  },
  {
    "id": 5,
    "description": "Resource Five",
    "createdTime": "2017-07-13T22:36:28.384",
    "modifiedTime": null
  },
  {
    "id": 6,
    "description": "Resource Six",
    "createdTime": "2017-07-13T22:36:28.384",
    "modifiedTime": null
  },
  {
    "id": 7,
    "description": "Resource Seven",
    "createdTime": "2017-07-13T22:36:28.384",
    "modifiedTime": null
  },
  {
    "id": 8,
    "description": "Resource Eight",
    "createdTime": "2017-07-13T22:36:28.384",
    "modifiedTime": null
  },
  {
    "id": 9,
    "description": "Resource Nine",
    "createdTime": "2017-07-13T22:36:28.384",
    "modifiedTime": null
  },
  {
    "id": 10,
    "description": "Resource Ten",
    "createdTime": "2017-07-13T22:36:28.384",
    "modifiedTime": null
  }
]
{% endhighlight %}

We can request for the same resource as `XML` representation:

{% highlight xml %}
> curl -i -H "Accept: application/xml" http://localhost:8080/api/v1.0/resources

HTTP/1.1 200 OK
Content-Type: application/xml
content-length: 1288
connection: keep-alive

<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<resources>
  <resource>
    <createdTime>2017-07-13T22:36:28.384</createdTime>
    <description>Resource One</description>
    <id>1</id>
  </resource>
  <resource>
    <createdTime>2017-07-13T22:36:28.384</createdTime>
    <description>Resource Two</description>
    <id>2</id>
  </resource>
  <resource>
    <createdTime>2017-07-13T22:36:28.384</createdTime>
    <description>Resource Three</description>
    <id>3</id>
  </resource>
  <resource>
    <createdTime>2017-07-13T22:36:28.384</createdTime>
    <description>Resource Four</description>
    <id>4</id>
  </resource>
  <resource>
    <createdTime>2017-07-13T22:36:28.384</createdTime>
    <description>Resource Five</description>
    <id>5</id>
  </resource>
  <resource>
    <createdTime>2017-07-13T22:36:28.384</createdTime>
    <description>Resource Six</description>
    <id>6</id>
  </resource>
  <resource>
    <createdTime>2017-07-13T22:36:28.384</createdTime>
    <description>Resource Seven</description>
    <id>7</id>
  </resource>
  <resource>
    <createdTime>2017-07-13T22:36:28.384</createdTime>
    <description>Resource Eight</description>
    <id>8</id>
  </resource>
  <resource>
    <createdTime>2017-07-13T22:36:28.384</createdTime>
    <description>Resource Nine</description>
    <id>9</id>
  </resource>
  <resource>
    <createdTime>2017-07-13T22:36:28.384</createdTime>
    <description>Resource Ten</description>
    <id>10</id>
  </resource>
</resources>
{% endhighlight %}

Let us write the REST operation for getting a specific resource `/api/v1.0/resources/{id}`:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/ResourceService.java' %}

{% highlight java %}
...
/**
 * GET /api/v1.0/resources/:id : get the resource specified by the identifier.
 * 
 * @param id the id to the resource being looked up
 * @return the {@code Resource} with status 200 (OK) and body or status 404 (NOT FOUND)
 */
@GET
@Path("{id: [0-9]+}")
public Resource getResource(@PathParam("id") Long id) {
    Resource resource = new Resource(id, null, null, null);

    int index = Collections.binarySearch(resources, resource, Comparator.comparing(Resource::getId));

    if (index >= 0)
        return resources.get(index);
    else
        throw new WebApplicationException(Response.Status.NOT_FOUND);
}
...
{% endhighlight %}

The `@Path` annotation takes a variable (denoted by `{` and `}`) passed by the client, which is interpreted by Jersey
in the `@PathParam` and cast to a `Long` as `id`. The `: [0-9]+` is a regular expression which constraint the client
to use only positive whole numbers otherwise return `404` to the client.  
If the client passes the path parameter in the format the server accepts, the `id` of the resource will be searched
from within the `static` resources field. If it exist return the response to the client or else return a `404`.

Test this resource by running:

{% highlight bash %}
> curl -i -H "Accept: application/json" http://localhost:8080/api/v1.0/resources/1

HTTP/1.1 200 OK
Content-Type: application/json
content-length: 96
connection: keep-alive

{
  "id":1,
  "description": "Resource One",
  "createdTime": "2017-07-14T23:55:18.76",
  "modifiedTime": null
}
{% endhighlight %}

Now let us write our `POST` method that creates a new resource:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/ResourceService.java' %}

{% highlight java %}
...
/**
 * POST /api/v1.0/resources : creates a new resource.
 * 
 * @param resource the resource being sent by the client as payload
 * @return the {@code Resource} with status 201 (CREATED) and no -content or status 
 *         400 (BAD REQUEST) if the resource does not contain an Id or status
 *         409 (CONFLICT) if the resource being created already exists in the list
 */
@POST
public Response createResource(Resource resource) {
  if (Objects.isNull(resource.getId()))
    throw new WebApplicationException(Response.Status.BAD_REQUEST);

  int index = Collections.binarySearch(resources, resource, Comparator.comparing(Resource::getId));

  if (index < 0) {
    resource.setCreatedTime(LocalDateTime.now());
    resources.add(resource);
    return Response
            .status(Response.Status.CREATED)
            .location(URI.create(String.format("/api/v1.0/resources/%s", resource.getId())))
            .build();
  } else
      throw new WebApplicationException(Response.Status.CONFLICT);
}
{% endhighlight %}

What is happening in the above snippet is a `POST` request that returns a `201` status code and a `Location` header
with the location of the newly created resource.  
If the resource being created already exists on the server an error code of `409` is returned by the server.

{% highlight bash %}
> curl -i -X POST -H "Content-Type: application/json" -d "{ \"id\": 87, \"description\": \"Resource Eighty-Seven\"}" http://localhost:8080/api/v1.0/resources

HTTP/1.1 201 Created
Location: http://localhost:8080/api/v1.0/resources/87
content-length: 0
connection: keep-alive
{% endhighlight %}

For those not using Windows, you should omit the escape `\`.

The remaining two methods of our webservice is shown below:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/ResourceService.java' %}

{% highlight java %}
...
/**
 * PUT /api/v1.0/resources/:id : update a resource identified by the given id
 * 
 * @param id the identifier of the resource to be updated
 * @param resource the resource that contains the update
 * @return the {@code Resource} with a status code 204 (NO CONTENT) or status code
 *         404 (NOT FOUND) when the resource being updated cannot be found
 */
@PUT
@Path("{id: [0-9]+}")
public Response updateResource(@PathParam("id") Long id, Resource resource) {
  resource.setId(id);
  int index = Collections.binarySearch(resources, resource, Comparator.comparing(Resource::getId));

  if (index >= 0){
    Resource updatedResource = resources.get(index);
    updatedResource.setModifiedTime(LocalDateTime.now());
    updatedResource.setDescription(resource.getDescription());
    resources.set(index, updatedResource);
    return Response
           .status(Response.Status.NO_CONTENT)
           .build();
  } else
    throw new WebApplicationException(Response.Status.NOT_FOUND);
}

/**
 * DELETE /api/v1.0/resources/:id : delete a resource identified by the given id
 * 
 * @param id the identifier of the resource to be deleted
 * @return the {@code Response} with a status code of 204 (NO CONTENT) or status code
 *         404 (NOT FOUND) when there is no resource with the given identifier
 */
@DELETE
@Path("{id: [0-9]+}")
public Response deleteResource(@PathParam("id") Long id) {
  Resource resource = new Resource(id, null, null, null);
  int index = Collections.binarySearch(resources, resource, Comparator.comparing(Resource::getId));

  if (index >= 0) {
  resources.remove(index);
  return Response
         .status(Response.Status.NO_CONTENT)
         .build();

  } else
      throw new WebApplicationException(Response.Status.NOT_FOUND);
}
{% endhighlight %}

Test the `PUT` endpoint with the following command:

{% highlight bash %}
> curl -i -X PUT -H "Content-Type: application/json" -d "{\"description\": \"Resource One Modified\"}" http://localhost:8080/api/v1.0/resources/1

HTTP/1.1 204 No Content
content-length: 0
connection: keep-alive
{% endhighlight %}

Test `DELETE` endpoint with the following command:

{% highlight bash %}
> curl -i -X DELETE  http://localhost:8080/api/v1.0/resources/1

HTTP/1.1 204 No Content
content-length: 0
connection: keep-alive
{% endhighlight %}

# Conclusion
In this post we built up on our previous post by wiring up the best of Spring and Jersey in the same application. In the next post we will learn how to [secure a RESTful
Web Service]({% post_url 2017-11-07-securing-a-spring-rest-service-with-jwt %}) using JWT.   
As usual you can find the full example to this guide {% include source.html %}. 
Until the next post, keep doing cool things :+1:.


[cURL]:                                https://curl.haxx.se/
[Initializr]:                          https://start.spring.io/
[JAX-RS]:                              https://jcp.org/en/jsr/detail?id=339
[JDK]:                                 http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Jersey]:                              http://jersey.github.io/
[Maven]:                               http://maven.apache.org
[Spring]:                              http://projects.spring.io/spring-framework/ 
