---
layout:     post
title:      Introduction to Microservices Using Spring-Boot
date:       2018-08-17 17:50:52 +0000
categories: blog
tags:       java maven spring-boot microservices
section:    blog
author:     juliuskrah
repo:       intro-microservice/tree/master
---
> Microservices enable different teams to work on smaller pieces, using independent technologies, producing safer, more frequent 
  deployments

# Introduction

If you are not already using Microservices, you are safely out of the early adopter phase of the adoption curve, and it is probably time
to get started. In this article, we will demonstrate the essential components for creating RESTful microservices, using `Consul` service 
registry, `Spring Boot` for all the scaffolding, dependency injection, and dependencies, `Maven` for the build, and Spring REST.

Over the last two decades, enterprise has become very agile in our SDLC process, but our applications tend to still be rather monolith, 
with huge jars supporting all of the varied APIs and versions in play. But nowadays there is a push towards more Lean, DevOps-ey 
processes, and functionality is becoming "serverless". Refactoring to Microservices can decouple code and resources, make builds smaller,
releases safer, and APIs more stable.


# Prerequisites

- [Java Development Kit][JDK]{:target="_blank"}
- [Maven][]{:target="_blank"}
- [Docker][]{:target="_blank"} (Optional)

# Purpose

In this post, we will build a simple geospatial application that clients can call with two spatial points. The `points` microservice will
send the points the client submitted to a `distance` microservice to calculate the distance between the given points, and then return 
the result to the points service, exposing all that via a rest call.

![Architecture](https://i.imgur.com/a5rQivV.png)
{:.image-caption}
*Microservice Architecture*

## Project Structure

At the end of this guide our folder structure will look similar to the following:
```
.
|__distance/
|  |__src/
|  |  |__main/
|  |  |  |__java/
|  |  |  |  |__com/
|  |  |  |  |  |__juliuskrah/
|  |  |  |  |  |  |__distance
|  |  |  |  |  |  |  |__DistanceApplication.java
|  |  |  |__resources/
|  |  |  |  |__application.yaml
|  |  |  |  |__bootstrap.yaml
|  |__pom.xml
|__points/
|  |__src/
|  |  |__main/
|  |  |  |__java/
|  |  |  |  |__com/
|  |  |  |  |  |__juliuskrah/
|  |  |  |  |  |  |__point/
|  |  |  |  |  |  |  |__PointsApplication.java
|  |  |  |__resources/
|  |  |  |  |__application.yaml
|  |  |  |  |__bootstrap.yaml
|  |__pom.xml
```

Before we get to work on creating our microservices, let's prepare our environment by setting up `Consul`.

## Download Consul Service Registry
We will be using Hashicorp Consul for service discovery, so head over to [https://www.consul.io/downloads.html](https://www.consul.io/downloads.html){:target="_blank"}
and download Consul, for Windows, Linux, Mac, etc. That will provide an executable that you will want to add to your path.

From a shell prompt, launch Consul in dev mode:

{% highlight posh %}
PS C:\> consul agent -dev
{% endhighlight %}

If you prefer to use docker, here's the command to start consul in dev mode:

{% highlight posh %}
PS C:\> docker run -d --name=dev-consul -e CONSUL_BIND_INTERFACE=eth0 consul
{% endhighlight %}

To verify that it is running, head over to a browser and access the consul UI [http://localhost:8500](http://localhost:8500){:target="_blank"}. 
If all is well, consul should report that it is alive and well.

![Consul Up](https://i.imgur.com/S2mPRhr.png)
{:.image-caption}
*Consul running on port 8500*

If there are any issues at this point, be sure ports `8500` and `8600` are available.

# Create the SpringBoot application

We will use [Spring Intitializr](https://start.spring.io/){:target="_blank"} to create the scaffolding for our SpringBoot applications.
Head over to the _Spring Initializr_ webapp and create the _distance microservice_. Select **Web** and **Consul Discovery** then
click _Generate Project_ to download a zip file. Extract this zip file and load the project into your favorite IDE or text editor.

![Distance Service](https://i.imgur.com/JUUH1N8.png)
{:.image-caption}
*Generate Distance Microservice*

We can use the generated `application.properties`, but Spring-Boot also recognizes `YAML` format, and that's a little easier to
visualize, so let's rename it `application.yaml`. Also create a `bootstrap.yaml` file.

We will name the microservice `distance`. We can specify a port or use port `0` to have the application use an available port. In our 
case we will use `57116`. If you deploy this service as a Docker container you will be able to map that to any port of your choosing. 
Let's name the application in `bootstrap.yaml` and specify our port by adding the following to our `application.yaml`:

file: {% include file-path.html file_path='distance/src/main/resources/application.yaml' %}

{% highlight yaml %}
server:
  port: 57116
spring:
  cloud:
    consul:
      discovery:
        register-health-check: false
{% endhighlight %}

file: {% include file-path.html file_path='distance/src/main/resources/bootstrap.yaml' %}

{% highlight yaml %}
spring:
  application:
    name: distance
{% endhighlight %}

Now build the project, and run it:

{% highlight posh %}
PS C:\> .\mvnw spring-boot:run
{% endhighlight %}

We should now see this service in Consul, so let's head back over to our browser, load up 
[http://localhost:8500/ui/dc1/services](http://localhost:8500/ui/dc1/services){:target="_blank"}
(or refresh if you're already there).

![Distance Consul](https://i.imgur.com/ifuSYBi.png)
{:.image-caption}
*Distance Service registered in Consul*

As you can see from the above diagram, the `distance` service has been registered with consul.

We need to add the `JTS` library to the application classpath to aid us with the distance measurement:

file: {% include file-path.html file_path='distance/pom.xml' %}

{% highlight xml %}
...
<dependency>
  <groupId>org.locationtech.jts</groupId>
  <artifactId>jts-core</artifactId>
  <version>1.15.1</version>
</dependency>
...
{% endhighlight %}

The `distance microservice` will serve one request `/distance/start/@{start}/dest/@{dest}` which returns the distance computed.

file: {% include file-path.html file_path='distance/src/main/java/com/juliuskrah/distance/DistanceApplication.java' %}

{% highlight java %}
@RestController
@SpringBootApplication
public class DistanceApplication {
  static GeometryFactory fact = new GeometryFactory();
  static WKTReader wktRdr = new WKTReader(fact);

  public static void main(String[] args) {
    SpringApplication.run(DistanceApplication.class, args);
  }
	
  @GetMapping(path = "/distance/start/@{start}/dest/@{dest}", produces = "application/json")
  public double points(@PathVariable String start, @PathVariable String dest) {
    double distance = 0.0;
    String[] pointA = start.split(",");
    String[] pointB = dest.split(",");
    String wktA = "POINT (" + pointA[0] + " " + pointA[1] +")";
    String wktB = "POINT (" + pointB[0] + " " + pointB[1] +")";
		
    try {
      Geometry A = wktRdr.read(wktA);
      Geometry B = wktRdr.read(wktB);

      System.out.println("Geometry A: " + A);
      System.out.println("Geometry B: " + B);
      DistanceOp distOp = new DistanceOp(A, B);
      distance = distOp.distance();
			
    } catch (ParseException e) {
      // ignore
    }	
    return distance;
  }	
}
{% endhighlight %}

Restart the distance service and test the endpoint `http://localhost:57116/distance/start/@50.8307467,-1.2099742/dest/@5.5790564,-0.7073872`,
and you should see `45.25448120020473`.

Our first microservice is open for business!

## Points Microservice

Head over to [Spring Intitializr](https://start.spring.io/){:target="_blank"} to create the `points microservice`. 
Select **Web** and **Consul Discovery**, change "Artifact" to _points_ then click _Generate Project_ to download a zip file. 
Extract the contents of the zip file and load the project into your favorite IDE or text editor.

We will name the microservice `points` in `bootstrap.yaml`. We will use port `57216` for the points microservice:

file: {% include file-path.html file_path='points/src/main/resources/application.yaml' %}

{% highlight yaml %}
server:
  port: 57216
spring:
  cloud:
    consul:
      discovery:
        register-health-check: false
{% endhighlight %}

file: {% include file-path.html file_path='points/src/main/resources/bootstrap.yaml' %}

{% highlight yaml %}
spring:
  application:
    name: points
{% endhighlight %}

Give it a spin `.\mvnw spring-boot:run` to ensure everything is working.

If it's all good we can implement the `points microservice`:

file: {% include file-path.html file_path='points/src/main/java/com/juliuskrah/point/PointsApplication.java' %}

{% highlight java %}
@RestController
@SpringBootApplication
public class PointsApplication {
  @Autowired
  private RestTemplate restTemplate;

  public static void main(String[] args) {
    SpringApplication.run(PointsApplication.class, args);
  }

  @GetMapping(path = "/points/start/@{start}/dest/@{dest}", produces = "application/json")
  public double points(@PathVariable String start, @PathVariable String dest) {
    return restTemplate.getForObject("http://distance/distance/start/@{start}/dest/@{dest}",
      double.class, start, dest);
  }

  @Bean
  @LoadBalanced
  public RestTemplate loadBalancedRestTemplate(RestTemplateBuilder builder) {
    return builder.build();
  }
}
{% endhighlight %}

A few things to note about the snippet above -- `@LoadBalanced`, annotation to mark a `RestTemplate` bean to be configured to use 
a round-robin `LoadBalancerClient` for client-side load-balancing. This same annotation allows `RestTemplate` to determine the URI
of the downstream microservice by name. This is how the `http://DISTANCE/distance/start/@{start}/dest/@{dest}` seemingly works despite
missing a `host` and `port` in the URI passed to `RestTemplate#getForObject`.

This approach is efficient as we leave the job of discovery the services' host and port to the discovery server (consul); we don't
have to worry about knowing the IPs of all microservices. In a large project consisting of thousands of microservices, you will soon
come to agree that knowing all IPs of services starts to become a jarring task; this is where service discovery starts to shine.

Let's put the `points microservice` to the test `http://localhost:57216/points/start/@5.8307467,-1.2099742/dest/@5.5790564,-0.7073872`.

This completes out points service using Spring-Boot.

# Conclusion
In this tutorial we learned the fundamentals of Microservices. We learned what is a Discovery client, we also built two microservices
to demonstrate how service discovery allows microservices to find each other.  
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.

[Docker]:                   https://www.docker.com/
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org
