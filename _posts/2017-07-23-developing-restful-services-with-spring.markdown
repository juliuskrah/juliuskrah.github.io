---
layout:     series
title:      Developing RESTful Services with Spring
date:       2017-07-23 12:08:50 +0000
categories: tutorial
tags:       java maven spring-boot rest
section:    series
author:     juliuskrah
repo:       rest-example/tree/spring
---
> The [Spring Framework][Spring]{:target="_blank"} has in  recent years emerged as a robust REST solution for Java 
  developers.

# Introduction
In the second part of this series we are going to explore the `Client-Server` REST constraint. This constraint 
postulates separation of concerns which allows the client and the server evolve independently. The client does not
need to worry about the server's implementation, provided the server's interface does not change.  
We will re-implement the server using the `Spring Framework`but  maintaining the interfaces from the 
[previous post]({% post_url 2017-07-16-developing-restful-services-with-jax-rs-jersey %}). The client will
continue to access it the same way without prior knowledge of the implementation.

> For a basic introduction to rest, checkout the 
  [first article]({% post_url 2017-07-16-developing-restful-services-with-jax-rs-jersey %}) in this series.

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
To follow along this guide, your development system should have the following setup:
- [Java Development Kit][JDK]{:target="_blank"}  
- [Maven][]{:target="_blank"}
- [cURL][]{:target="_blank"}
- [Heroku Account][Heroku]{:target="_blank"}
- [Heroku CLI][]{:target="_blank"}
- [Git][]{:target="_blank"}

# Creating Project Template
Head over to the [Spring Initializr][Initializr]{:target="_blank"} website to generate a Spring project template:

![spring.io](https://i.imgur.com/4iKKqY8.png)

{:.image-caption}
*Spring Initializr*

Select `Web` and `DevTools` as dependencies and generate the project. `DevTools` is a handy tool to have during 
development as it offers _live reload_ when code changes.

Download and extract the template and let's get to work :smile:.

## Building Resources
Before we proceed, run the generated project to ensure it works:

{% highlight bash %}
mvnw clean spring-boot:run   # Windows
./mvnw clean spring-boot:run # Linux, Mac
{% endhighlight %}

If everything goes well, `Tomcat` should be started on port `8080` and wait for `http` requests.

## Setting up dependencies
To build a RESTful webservice with Spring, and enable `XML` representation we can add the 
`jackson-datatype-xml` to our `pom.xml`:

file: {% include file-path.html file_path='pom.xml' %}

{% highlight xml %}
<dependency>
  <groupId>com.fasterxml.jackson.dataformat</groupId>
  <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
<dependency>
  <groupId>org.codehaus.woodstox</groupId>
  <artifactId>woodstox-core-asl</artifactId>
  <version>4.4.1</version>
</dependency>
{% endhighlight %}

## Building Resources
We will create a `POJO` to represent our REST resource.

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/Resource.java' %}

{% highlight java %}
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
@RestController
@RequestMapping(path = "/api/v1.0/resources", produces = {
        MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE })
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

    @GetMapping
    public List<Resource> getResources() {
        return resources;
    }
    ...
}
{% endhighlight %}

> For a production ready application, you will normally connect to a database. For the purpose of this tutorial,
  we will use a `static` field to initialize our list.

In the `ResourceService` we have specified the root context path (`/api/v1.0/resources`) we are going to access the
service.  
In the same service class we have also created a `GetMapping` endpoint which returns a list of all resources available
on the server. The resource will be represented as `JSON` or `XML` identified by 
`produces = { MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE }`.  
The [`@RestController`][RestController]{:target="_blank"} annotation indicate to Spring method return value should be
bound to the [web response body](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/ResponseBody.html){:target="_blank"}.

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
@GetMapping("{id:[0-9]+}")
public ResponseEntity<Resource> getResource(@PathVariable Long id) {
  Resource resource = new Resource(id, null, null, null);

  int index = Collections.binarySearch(resources, resource, Comparator.comparing(Resource::getId));

  if (index >= 0)
    return ResponseEntity.ok(resources.get(index));
  else
    return ResponseEntity.notFound().build();
}
{% endhighlight %}

The `@GetMapping` annotation takes a variable (denoted by `{` and `}`) passed by the client, which is converted by 
Spring to a `Long` using automatic type conversion.
The `:[0-9]+` is a regular expression which constraint the client to use only positive whole numbers otherwise 
the server returns `404` to the client.  
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
@PostMapping
public ResponseEntity<Void> createResource(@RequestBody Resource resource, UriComponentsBuilder b) {
  if (Objects.isNull(resource.getId()))
    return ResponseEntity.badRequest().build();

  int index = Collections.binarySearch(resources, resource, Comparator.comparing(Resource::getId));

  if (index < 0) {
    resource.setCreatedTime(LocalDateTime.now());
    resources.add(resource);
    UriComponents uriComponents = b.path("/api/v1.0/resources/{id}").buildAndExpand(resource.getId());
    return ResponseEntity.created(uriComponents.toUri()).build();
  } else
    return ResponseEntity.status(HttpStatus.CONFLICT).build();
}
{% endhighlight %}

What is happening in the above snippet is a `PostMapping` request that returns a `201` (created) status code and a 
`Location` header with the location of the newly created resource.  
If the resource being created already exists on the server an error code of `409` (conflict) is returned by the 
server.

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
@PutMapping("{id:[0-9]+}")
public ResponseEntity<Void> updateResource(@PathVariable Long id, @RequestBody Resource resource) {
  resource.setId(id);

  int index = Collections.binarySearch(resources, resource, Comparator.comparing(Resource::getId));

  if (index >= 0) {
    Resource updatedResource = resources.get(index);
    updatedResource.setModifiedTime(LocalDateTime.now());
    updatedResource.setDescription(resource.getDescription());
    resources.set(index, updatedResource);
    return ResponseEntity.noContent().build();
  } else
    return ResponseEntity.notFound().build();
}

@DeleteMapping("{id:[0-9]+}")
public ResponseEntity<Void> deleteResource(@PathVariable Long id) {
  Resource resource = new Resource(id, null, null, null);

  int index = Collections.binarySearch(resources, resource, Comparator.comparing(Resource::getId));

  if (index >= 0) {
    resources.remove(index);
    return ResponseEntity.noContent().build();
  } else
    return ResponseEntity.notFound().build();
}
{% endhighlight %}

Test the `Put` endpoint with the following command:

{% highlight bash %}
> curl -i -X PUT -H "Content-Type: application/json" -d "{\"description\": \"Resource One Modified\"}" http://localhost:8080/api/v1.0/resources/1

HTTP/1.1 204 No Content
content-length: 0
connection: keep-alive
{% endhighlight %}

Test `Delete` endpoint with the following command:

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
Creating app... done, floating-gorge-84071
https://floating-gorge-84071.herokuapp.com/ | https://git.heroku.com/floating-gorge-84071.git
{% endhighlight %}

This creates a heroku app called `floating-gorge-84071`. Heroku defines an environment variable `$PORT` which is
the HTTP port exposed over firewall for HTTP traffic.

Next create a [`Procfile`][Procfile]{:target="_blank"}. This is a plain text file called `Procfile` not `procfile` or 
`procfile.txt` but just `Procfile`.  
In this file we will create a `web` process type. The `web` process type is a special process type that listens for
HTTP traffic.

file: {% include file-path.html file_path='Procfile' %}

```
web: java -jar target/*.war --server.port=$PORT
```

The application is ready to be deployed to Heroku with `git add . && git push heroku master`. But first let us
test the application locally to make sure everything works. 

{% highlight bash %}
> mvn clean package
> heroku local web
{% endhighlight %}

> For those on Windows platform, create a file `Procfile.windows` for local testing.

file: {% include file-path.html file_path='Procfile.windows' %}

```
web: java -jar target\rest-service-0.0.1-SNAPSHOT.war --server.port=${PORT:5000}
```

Next run (Windows only):

{% highlight bash %}
> mvn clean package
> heroku local web -f Procfile.windows
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
remote:        https://floating-gorge-84071.herokuapp.com/ deployed to Heroku
remote:
remote: Verifying deploy... done.
To https://git.heroku.com/floating-gorge-84071.git
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

You can view the logs for the application by running this command:

{% highlight bash %}
> heroku logs --tail
2017-08-05T22:12:53.118407+00:00 app[web.1]:  =========|_|==============|___/=/_/_/_/
2017-08-05T22:12:53.120445+00:00 app[web.1]:  :: Spring Boot ::        (v1.5.6.RELEASE)
2017-08-05T22:12:53.120470+00:00 app[web.1]:
2017-08-05T22:12:53.118202+00:00 app[web.1]:
2017-08-05T22:12:53.118216+00:00 app[web.1]:   .   ____          _            __ _ _
2017-08-05T22:12:53.118258+00:00 app[web.1]:  /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
2017-08-05T22:12:53.118290+00:00 app[web.1]: ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
2017-08-05T22:12:53.118325+00:00 app[web.1]:  \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
2017-08-05T22:12:53.118367+00:00 app[web.1]:   '  |____| .__|_| |_|_| |_\__, | / / / /
{% endhighlight %}

# Conclusion
In this post we focused on the `Client-Server` constraint of REST. We learned how to implement REST with Spring.
We also learned how to deploy a Spring-Boot app to Heroku.    
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.


[cURL]:                                https://curl.haxx.se/
[Git]:                                 https://git-scm.com/
[Heroku]:                              https://signup.heroku.com/
[Heroku CLI]:                          https://devcenter.heroku.com/articles/getting-started-with-java#set-up
[Initializr]:                          https://start.spring.io/
[JDK]:                                 http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                               http://maven.apache.org
[Procfile]:                            https://devcenter.heroku.com/articles/procfile
[RestController]:                      https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/RestController.html
[Spring]:                              http://projects.spring.io/spring-framework/ 
