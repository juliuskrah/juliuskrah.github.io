---
layout:     series
title:      Developing RESTful Services with JAX-RS (Jersey)
date:       2017-07-16 00:08:10 +0000
categories: tutorial
tags:       java maven jersey jax-rs rest javaee
section:    series
author:     juliuskrah
repo:       rest-example/tree/jersey
---
> The REST architectural style describes six constraints. These constraints, applied to the architecture, were 
  originally communicated by Roy Fielding in his [doctoral dissertation][Fielding Dissertation]{:target="_blank"} 
  and defines the basis of RESTful-style.

# Introduction
REST stands for REpresentational State Transfer. The REST architectural style describes six constraints. These 
constraints, applied to the architecture, were originally communicated by Roy Fielding in his doctoral dissertation
and defines the basis of RESTful-style.

**The six constraints are: (click on the constraint below to read more)**

- [Uniform Interface][]{:target="_blank"}
- [Stateless][]{:target="_blank"}
- [Cacheable][]{:target="_blank"}
- [Client-Server][]{:target="_blank"}
- [Layered System][]{:target="_blank"}
- [Code on Demand (optional)][Code on Demand]{:target="_blank"}

[JAX-RS][]{:target="_blank"} (JSR-339) as a specification defines a set of Java APIs for the development of Web 
services built according to the REpresentational State Transfer (REST) architectural style.

[Jersey RESTful][Jersey]{:target="_blank"} Web Services framework is an open source, production quality, framework 
for developing RESTful Web Services in Java that provides support for JAX-RS APIs and serves as a JAX-RS 
([JSR 311][]{:target="_blank"} & [JSR 339][]{:target="_blank"}) [Reference Implementation][RI]{:target="_blank"}.


## Project Structure
At the end of this guide our folder structure will look similar to the following:

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__juliuskrah
|  |  |  |  |  |__App.java
|  |  |  |  |  |__Resource.java
|  |  |  |  |  |__ResourceService.java
|__pom.xml
```


# Prerequisites
To follow along this guide, you should have the following set up:

- [Java Development Kit][JDK]{:target="_blank"}  
- [Maven][]{:target="_blank"}

_Optional_

- [cURL][]{:target="_blank"}
- [Heroku Account][Heroku]{:target="_blank"}
- [Heroku CLI][]{:target="_blank"}
- [Git][]{:target="_blank"}


# Creating Project Template
To create the project structure, we would use the `Maven Archetype` to generate a project skeleton. In the directory
where you want to generate your project, run the following commands:

{% highlight bash %}
mvn archetype:generate
{% endhighlight %}

You will get a list of archetypes to select from. Select `1002: org.apache.maven.archetypes:maven-archetype-quickstart (An archetype which contains a sample Maven project.)`
maven quickstart archetype. This is a simple maven project with just one class.

{% highlight bash %}
...
Choose a number or apply filter (format: [groupId:]artifactId, case sensitive contains): 1002:
{% endhighlight %}

Type `1002` and hit enter.

{% highlight bash %}
...
Choose org.apache.maven.archetypes:maven-archetype-quickstart version:
1: 1.0-alpha-1
2: 1.0-alpha-2
3: 1.0-alpha-3
4: 1.0-alpha-4
5: 1.0
6: 1.1
Choose a number: 6:
{% endhighlight %}

Hit enter to accept the default of `1.1`.

{% highlight bash %}
...
Define value for property 'groupId':
{% endhighlight %}

Type group Id e.g. `com.juliuskrah` and hit enter

{% highlight bash %}
...
Define value for property 'artifactId':
{% endhighlight %}

Type artifact Id e.g. `rest-example` and hit enter

{% highlight bash %}
...
Define value for property 'version' 1.0-SNAPSHOT: :
{% endhighlight %}

Hit enter to accept `1.0-SNAPSHOT` and proceed

{% highlight bash %}
...
Define value for property 'package' com.juliuskrah: :
{% endhighlight %}

Hit enter to accept default and proceed

{% highlight bash %}
...
Confirm properties configuration:
groupId: com.juliuskrah
artifactId: rest-example
version: 1.0-SNAPSHOT
package: com.juliuskrah
 Y: :
{% endhighlight %}

Review your selection and hit enter to proceed. At this stage your project is generated in `rest-example`.

## Setting up dependencies
To build a RESTful webservice with Jersey, we must add the Jersey dependencies to our `pom.xml`:

file: {% include file-path.html file_path='pom.xml' %}

{% highlight xml %}
...
<dependencyManagement>
  <dependencies>
     <dependency>
       <groupId>org.glassfish.jersey</groupId>
       <artifactId>jersey-bom</artifactId>
       <version>2.26-b07</version>
       <type>pom</type>
       <scope>import</scope>
     </dependency>
   </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>org.glassfish.jersey.containers</groupId>
    <artifactId>jersey-container-netty-http</artifactId>
  </dependency>
  <dependency>
    <groupId>org.glassfish.jersey.inject</groupId>
    <artifactId>jersey-hk2</artifactId>
  </dependency>
  <dependency>
    <groupId>org.glassfish.jersey.media</groupId>
    <artifactId>jersey-media-json-jackson</artifactId>
  </dependency>
</dependencies>
{% endhighlight %}

In the snippet above we have specified the `dependencyManagement` so that we can omit the version of 
`jersey-container-netty-http`.

# What we will Create

| **HTTP Method**    | **URI**                              | **Action**                 |
|--------------------|:-------------------------------------|:---------------------------|
|GET                 | `/api/v1.0/resources/`               |Retrieve list of resources  |
|GET                 | `/api/v1.0/resources/[resource_id]`  |Retrieve a resource         |
|POST                | `/api/v1.0/resources/`               |Create a new resource       |
|PUT                 | `/api/v1.0/resources/[resource_id]`  |Update an existing resource |
|DELETE              | `/api/v1.0/resources/[resource_id]`  |Delete a resource           |
{:.comparison}


The HTTP GET method is used to **read** (or retrieve) a representation of a resource. In the `“happy”` (or non-error) 
path, GET returns a representation in `XML` or `JSON` and an HTTP response code of `200` (OK). In an error case, it 
most often returns a `404` (NOT FOUND) or `400` (BAD REQUEST).  
According to the design of the HTTP specification, GET (along with HEAD) requests are used only to read data and not 
change it. Therefore, when used this way, they are considered safe. That is, they can be called without risk of data 
modification or corruption — calling it once has the same effect as calling it 10 times, or none at all. Additionally,
GET (and HEAD) is idempotent, which means that making multiple identical requests ends up having the same result as a
single request.  
Do not expose unsafe operations via GET — it should never modify any resources on the server.

**Examples:**
- GET http://www.example.com/resources/1  
- GET http://www.example.com/resources/1/items

---

The POST verb is most-often utilized to **create** new resources. In particular, it's used to create subordinate 
resources. That is, subordinate to some other (e.g. parent) resource. In other words, when creating a new resource, 
POST to the parent and the service takes care of associating the new resource with the parent, assigning an ID (new 
resource URI), etc.  
On successful creation, return HTTP status `201`, returning a `Location` header with a link to the newly-created 
resource with the 201 HTTP status.  
POST is neither safe nor idempotent. It is therefore recommended for non-idempotent resource requests. Making two 
identical POST requests will most-likely result in two resources containing the same information.

**Examples:**
- POST http://www.example.com/resources  
- POST http://www.example.com/resources/1/items

---

PUT is most-often utilized for **update** capabilities, PUT-ing to a known resource URI with the request body 
containing the newly-updated representation of the original resource.  
However, PUT can also be used to create a resource in the case where the resource ID is chosen by the client instead 
of by the server. In other words, if the PUT is to a URI that contains the value of a non-existent resource ID. Again,
the request body contains a resource representation. Many feel this is convoluted and confusing. Consequently, this 
method of creation should be used sparingly, if at all.  
Alternatively, use POST to create new resources and provide the client-defined ID in the body 
representation—presumably to a URI that doesn't include the ID of the resource (see POST above).
On successful update, return `200` (or `204` if not returning any content in the body) from a PUT. If using PUT for 
create, return HTTP status `201` on successful creation. A body in the response is optional — providing one consumes 
more bandwidth. It is not necessary to return a link via a Location header in the creation case since the client 
already set the resource ID.  
PUT is not a safe operation, in that it modifies (or creates) state on the server, but it is idempotent. In other 
words, if you create or update a resource using PUT and then make that same call again, the resource is still there 
and still has the same state as it did with the first call.  
If, for instance, calling PUT on a resource increments a counter within the resource, the call is no longer 
idempotent. Sometimes that happens and it may be enough to document that the call is not idempotent. However, it's 
recommended to keep PUT requests idempotent. It is strongly recommended to use POST for non-idempotent requests.

**Examples:**
- PUT http://www.example.com/resources/1  
- PUT http://www.example.com/resources/1/items/1

---

DELETE is pretty easy to understand. It is used to **delete** a resource identified by a URI.  
On successful deletion, return HTTP status `200` (OK) along with a response body, perhaps the representation of the
deleted item (often demands too much bandwidth), or a wrapped response. Either that or return HTTP status `204` (NO
CONTENT) with no response body. In other words, a `204` status with no body, or the JSEND-style response and HTTP
status `200` are the recommended responses.  
HTTP-spec-wise, DELETE operations are idempotent. If you DELETE a resource, it's removed. Repeatedly calling DELETE 
on that resource ends up the same: the resource is gone. If calling DELETE say, decrements a counter (within the
resource), the DELETE call is no longer idempotent. As mentioned previously, usage statistics and measurements may be
updated while still considering the service idempotent as long as no resource data is changed. Using POST for 
non-idempotent resource requests is recommended.  
There is a caveat about DELETE idempotence, however. Calling DELETE on a resource a second time will often return a 
`404` (NOT FOUND) since it was already removed and therefore is no longer findable. This, by some opinions, makes 
DELETE operations no longer idempotent, however, the end-state of the resource is the same. Returning a 404 is 
acceptable and communicates accurately the status of the call.

**Examples:**
- DELETE http://www.example.com/resources/1  
- DELETE http://www.example.com/resources/1/items

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

The `Resource POJO` above consists of self explanatory feilds. The `@XmlRootElement` is put on the class to instruct
jersey on how to represent the resource as XML.  
The next thing is to wire up a `ResourceService`.

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/ResourceService.java' %}

{% highlight java %}
@Path("/api/v1.0/resources")
@Produces({ MediaType.APPLICATION_JSON, MediaType.APPLICATION_XML })
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

> In an actual production ready application, you will connect to a database. For the purpose of this tutorial,
  we will use a `static` list.

In the `ResourceService` we have specified the root context path we are going to access the service 
(`/api/v1.0/resources`).  
In the same service class we have also created a `GET` endpoint which returns a list of all resources available
on the server. The resource will be represented as `JSON` or `XML` identified by
`@Produces({ MediaType.APPLICATION_JSON, MediaType.APPLICATION_XML })`.

We will setup a [`Netty`][Netty]{:target="_blank"} server to serve our requests.

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/App.java' %}

{% highlight java %}
public class App {
  public static void main(String... cmd) throws IOException, InterruptedException {
    URI baseUri = UriBuilder.fromUri("http://localhost/").port(8080).build();
    ResourceConfig resourceConfig = new ResourceConfig().packages("com.juliuskrah");
    Channel server = NettyHttpContainerProvider.createServer(baseUri, resourceConfig, false);
    System.out.println("Press ENTER to terminate...");
    System.in.read();
    server.close().await();
  }
}
{% endhighlight %}

With the server setup we can test our REST resource. Start the server by running the following command:

{% highlight bash %}
mvn clean compile exec:java -Dexec.mainClass="com.juliuskrah.App"
{% endhighlight %}

With the Netty server up and running, open another shell window and execute the following `cURL` command:

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

We can request for the same resource represented as `XML`:

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

Let us write the REST operation for getting a specific resource `/api/v1.0/resources/[resource_id]`:

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

    // Search the Resource list for a resource with the given id and return its index in the list
    int index = Collections.binarySearch(resources, resource, Comparator.comparing(Resource::getId));

    if (index >= 0)
        return resources.get(index);
    else
        throw new WebApplicationException(Response.Status.NOT_FOUND);
}
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

# Deploying to Heroku
To deploy this application to Heroku, we must ensure we have a Heroku account and `Heroku CLI`. Navigate to 
the root directory of your application and execute the following command from your terminal:

{% highlight bash %}
> heroku create
Creating app... done, murmuring-shore-59920
https://murmuring-shore-59920.herokuapp.com/ | https://git.heroku.com/murmuring-shore-59920.git
{% endhighlight %}

This creates a heroku app called `murmuring-shore-59920`. Heroku defines an environment variable `$PORT` which is
the HTTP port exposed over firewall for HTTP traffic. We will modify our source file to read this variable:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/App.java' %}

{% highlight java %}
public static void main(String... cmd) throws IOException, InterruptedException {
  int port = Objects.nonNull(System.getenv("PORT")) ? 
    Integer.valueOf(System.getenv("PORT")) : 8080;

  URI baseUri = UriBuilder.fromUri("http://localhost/").port(port).build();
  ResourceConfig resourceConfig = new ResourceConfig().packages("com.juliuskrah");
  NettyHttpContainerProvider.createServer(baseUri, resourceConfig, false);
  System.out.printf("Application running on %s\n", baseUri.toURL().toExternalForm());
}
{% endhighlight %}

Heroku needs to be able to run our web service with all the dependencies. We add the maven dependency plugin to our
project:

file: {% include file-path.html file_path='pom.xml' %}

{% highlight xml %}
...
<plugins>
  <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <executions>
      <execution>
        <id>copy-dependencies</id>
        <phase>package</phase>
        <goals>
          <goal>copy-dependencies</goal>
        </goals>
        <configuration>
          <includeScope>compile</includeScope>
        </configuration>
      </execution>
    </executions>
  </plugin>
</plugins>
{% endhighlight %}

Next create a [`Procfile`][Procfile]{:target="_blank"}. This is a plain text file called `Procfile` not `procfile` or 
`procfile.txt` but just `Procfile`.  
In this file we will create a `web` process type. The `web` process type is a special process type that listens for
HTTP traffic.

file: {% include file-path.html file_path='Procfile' %}

```
web: java -cp target/classes:target/dependency/* com.juliuskrah.App
```

The application is ready to be deployed to Heroku with `git add . && git push heroku master`. But first let us
test the application locally to make sure everything works. To test locally create an `.env` file:

file: {% include file-path.html file_path='.env' %}

```
PORT=2222
```

This creates an environment variable for local testing:

{% highlight bash %}
> mvn clean package
> heroku local web
{% endhighlight %}

Now that we are confident everything works, we will deploy to heroku:

{% highlight bash %}
> git add .
> git commit -am "Added heroku app"
> git push heroku master
Counting objects: 137, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (93/93), done.
Writing objects: 100% (137/137), 72.61 KiB | 0 bytes/s, done.
Total 137 (delta 42), reused 0 (delta 0)
remote: Compressing source files... done.
remote: Building source:
remote:
remote: -----> Java app detected
remote: -----> Installing OpenJDK 1.8... done
remote: -----> Executing: mvn -DskipTests clean dependency:list install
...
remote:        [INFO] ------------------------------------------------------------------------
remote:        [INFO] BUILD SUCCESS
remote:        [INFO] ------------------------------------------------------------------------
remote:        [INFO] Total time: 13.305 s
remote:        [INFO] Finished at: 2017-08-01T23:39:57Z
remote:        [INFO] Final Memory: 31M/342M
remote:        [INFO] ------------------------------------------------------------------------
remote: -----> Discovering process types
remote:        Procfile declares types -> web
remote:
remote: -----> Compressing...
remote:        Done: 82.4M
remote: -----> Launching...
remote:        Released v3
remote:        https://murmuring-shore-59920.herokuapp.com/ deployed to Heroku
remote:
remote: Verifying deploy... done.
To https://git.heroku.com/murmuring-shore-59920.git
 * [new branch]      origin -> master
{% endhighlight %}

We scale up a [dyno](https://devcenter.heroku.com/articles/dyno-types){:target="_blank"} with the following command:

{% highlight bash %}
> heroku ps:scale web=1
Scaling dynos... done, now running web at 1:Free
> heroku open
{% endhighlight %}

The last command opens the heroku app in your default browser. If you get a `404`, that is normal because you
haven't mapped any resource to `/`. Just append the url with `/api/v1.0/resources` to start hacking.


# Conclusion
In this post we learned REST and touched a little on the constraints. We learned how it is `HTTP` based.
We also talked about the basic `methods` in HTTP and how to leverage them in a RESTful application. We also learned
how to deploy a REST application to Heroku.    
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.

[Cacheable]:                http://restcookbook.com/Basics/caching/
[Client-Server]: https://github.com/brettshollenberger/codecabulary/blob/master/REST%20Constraint%20-%20Client-Server%20Separation.md
[Code on Demand]:           http://restfulapi.net/rest-architectural-constraints/#code-on-demand
[cURL]:                     https://curl.haxx.se/
[Fielding Dissertation]:    https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
[Git]:                      https://git-scm.com/
[Heroku]:                   https://signup.heroku.com/
[Heroku CLI]:               https://devcenter.heroku.com/articles/getting-started-with-java#set-up
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[JAX-RS]:                   https://jcp.org/en/jsr/detail?id=339
[Jersey]:                   http://jersey.github.io/
[JSR 311]:                  https://www.jcp.org/en/jsr/detail?id=311
[JSR 339]:                  https://www.jcp.org/en/jsr/detail?id=339
[Layered System]:           http://restfulapi.net/rest-architectural-constraints/#layered-system
[Maven]:                    http://maven.apache.org
[Netty]:                    http://netty.io/
[Procfile]:                 https://devcenter.heroku.com/articles/procfile
[RI]:                       https://en.wikipedia.org/wiki/Reference_implementation
[Stateless]:                https://stackoverflow.com/a/3105337/6157880
[Uniform Interface]:        https://stackoverflow.com/a/26049761/6157880
