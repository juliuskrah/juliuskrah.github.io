---
layout:     post
title:      'Spring: Integrating Tiles'
date:       2017-01-21 21:25:22 +0000
categories: blog
tags:       java spring tiles maven
section:    blog
author:     Julius Krah
repo:       spring-tiles-example
---
> [Apache Tiles][Tiles]{:target="_blank"} is a free open-sourced templating framework for modern Java applications. Based upon the 
  `Composite Pattern`, it is built to simplify the development of user interfaces. There are several integrations to the Tiles 
  templating framework.  
  [Spring][]{:target="_blank"} provides one such integration into the `Tiles` framework using `TilesConfigurer` and `TilesViewResolver`
  which internally registers `TilesView`.

# Introduction
In this post we are going to learn how to make `Spring` play nice with `Tiles`. If you are new to Spring, take a look at my
[previous post]({% post_url 2016-12-10-introduction-to-spring-and-dependency-injection-jsr-330 %}) for an introduction. If you are 
new to Tiles as well, look [here]({% post_url 2016-10-17-getting-started-with-tiles %}) for a beginner's overview.

## Pre-requisites
- [Java Development Kit][JDK]{:target="_blank"}  
- [Maven][]{:target="_blank"}

## Project Structure
To demonstrate the integration of Spring with Tiles, we will create a [Maven][]{:target="_blank"} project using the standard directory
structure as illustrated below:

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__jipasoft/
|  |  |  |  |  |__tiles/
|  |  |  |  |  |  |__RootConfig.java
|  |  |  |  |  |  |__config/
|  |  |  |  |  |  |  |__SpringConfig.java
|  |  |  |  |  |  |__initializer/
|  |  |  |  |  |  |  |__WebInitializer.java
|  |  |  |  |  |  |__web/
|  |  |  |  |  |  |  |__IndexController.java
|  |  |__resources/
|  |  |  |__log4j.properties
|  |__test/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__jipasoft/
|  |  |  |  |  |__tiles/
|  |  |  |  |  |  |__config/
|  |  |  |  |  |  |  |__AbstractTest.java
|  |  |  |  |  |  |  |__AppConfig.java
|  |  |__resources/
|  |  |  |__log4j.properties
|  |  |__webapp/
|  |  |  |__layouts/
|  |  |  |  |__classic.jsp
|  |  |  |__tiles/
|  |  |  |  |__banner.jsp
|  |  |  |  |__blog_header.jsp
|  |  |  |  |__common_menu.jsp
|  |  |  |  |__credits.jsp
|  |  |  |  |__home_body.jsp
|  |  |  |  |__navigation.jsp
|  |  |  |__WEB-INF/
|  |  |  |  |__tiles.xml
|__pom.xml
```

# Getting Dependencies
Using Maven, it is easier to get and manage dependencies for this introductory project. We will add the following dependencies to
our `pom.xml`:

{% highlight xml %}
<dependencies>
  ...
<!-- Apache Tiles -->
  <dependency>
    <groupId>org.apache.tiles</groupId>
    <artifactId>tiles-jsp</artifactId>
    <version>3.0.7</version>
  </dependency>
<!-- Servlet 3.1.0 and JSTL -->
  <dependency>
    <groupId>javax.servlet.jsp.jstl</groupId>
    <artifactId>jstl-api</artifactId>
    <version>1.2</version>
    <exclusions>
      <exclusion>
        <groupId>javax.servlet</groupId>
        <artifactId>servlet-api</artifactId>
      </exclusion>
    </exclusions>
  </dependency>
  <dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.1.0</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>javax.servlet.jsp</groupId>
    <artifactId>jsp-api</artifactId>
    <version>2.0</version>
  </dependency>
<!-- Spring -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>${spring.framework.version}</version>
  </dependency>
  ...
</dependencies>
{% endhighlight %}

Apart from the `Spring` and `Tiles` dependencies, we declare additional dependencies on the `Servlet` API and `JSP`. This is because
we will be running in a web environment and also require `JSP` for templating.

# Setting up Tiles
Before we bootstrap Spring for our tiles-based view resolver, let us create our layout template:

- [`src/main/webapp/layouts/classic.jsp`]({{ site.github.url }}/{{ site.github_username }}/{{ page.repo }}/tree/master/src/main/webapp/layouts/classic.jsp){:target="_blank"}  

Next we will create our Tiles attributes:

- [`src/main/webapp/tiles/banner.jsp`]({{ site.github.url }}/{{ site.github_username }}/{{ page.repo }}/tree/master/src/main/webapp/tiles/banner.jsp){:target="_blank"}  
- [`src/main/webapp/tiles/blog_header.jsp`]({{ site.github.url }}/{{ site.github_username }}/{{ page.repo }}/tree/master/src/main/webapp/tiles/blog_header.jsp){:target="_blank"}  
- [`src/main/webapp/tiles/common_menu.jsp`]({{ site.github.url }}/{{ site.github_username }}/{{ page.repo }}/tree/master/src/main/webapp/tiles/common_menu.jsp){:target="_blank"}  
- [`src/main/webapp/tiles/credits.jsp`]({{ site.github.url }}/{{ site.github_username }}/{{ page.repo }}/tree/master/src/main/webapp/tiles/credits.jsp){:target="_blank"}  
- [`src/main/webapp/tiles/home_body.jsp`]({{ site.github.url }}/{{ site.github_username }}/{{ page.repo }}/tree/master/src/main/webapp/tiles/home_body.jsp){:target="_blank"}  
- [`src/main/webapp/tiles/navigation.jsp`]({{ site.github.url }}/{{ site.github_username }}/{{ page.repo }}/tree/master/src/main/webapp/tiles/navigation.jsp){:target="_blank"}  

Finally we will create the `Tiles` definition XML file.  
`src/main/webapp/WEB-INF/tiles.xml`:

{% highlight xml %}
<tiles-definitions>
  <definition name="index" template="/layouts/classic.jsp">
    <put-attribute name="title" value="Home || Spring Tiles" />
    <put-attribute name="header" value="/tiles/banner.jsp" />
    <put-attribute name="menu" value="/tiles/common_menu.jsp" />
    <put-attribute name="body" value="/tiles/home_body.jsp" />
    <put-attribute name="footer" value="/tiles/credits.jsp" />
    <put-attribute name="heading" value="/tiles/blog_header.jsp" />
    <put-attribute name="navigation" value="/tiles/navigation.jsp" />
  </definition>
</tiles-definitions>
{% endhighlight %}

> **NOTE**  
  Refer to my previous [post]({% post_url 2016-10-17-getting-started-with-tiles %}) on `Tiles` if you are new to the Tiles framework.

# Creating Springbeans
In this section we will learn how to create Springbeans to wire it all together. We will configure Spring's `ApplicationContext`
in a web environment.  
`src/main/java/com/jipasoft/tiles/config/SpringConfig.java`:

{% highlight java %}
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.ViewResolver;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;
import org.springframework.web.servlet.view.UrlBasedViewResolver;
import org.springframework.web.servlet.view.tiles3.TilesConfigurer;
import org.springframework.web.servlet.view.tiles3.TilesView;

@Configuration
@EnableWebMvc
@ComponentScan({ "com.jipasoft.tiles.web" })
public class SpringConfig extends WebMvcConfigurerAdapter {

  @Override
  public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/resources/**").addResourceLocations("/resources/");
  }

  @Bean
  public ViewResolver tilesViewResolver() {
    TilesViewResolver resolver = new TilesViewResolver();

    return resolver;
  }

  @Bean
  public TilesConfigurer tilesConfigurer() {
    TilesConfigurer configurer = new TilesConfigurer();
    configurer.setDefinitions("/WEB-INF/tiles.xml");

    return configurer;
  }

}
{% endhighlight %}

In the above code snippet, we have defined two beans: `ViewResolver` and `TilesConfigurer`. The ViewResolver bean is used to resolve
views by name for a Spring `MVC` application. TilesConfigurer configures Tiles 3.x for the Spring Framework. In this bean we tell
Spring where to find `Tiles definitions` file(s) through the `TilesConfigurer#setDefinitions` method. This is absolutely optional if your
Tiles definition XML file is `/WEB-INF/tiles.xml`.  
Now we will bootstrap a controller to serve our Tiles views.  
`src/main/java/com/jipasoft/tiles/web/IndexController.java`:

{% highlight java linenos %}
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class IndexController {

  @GetMapping("/")
  public String home() {
    return "index";
  }

}
{% endhighlight %}

In the snippet above on line `4` is `Controller` which is a Spring `Component` instructing the framework to dispatch requests. The 
`GetMapping` on line `7` tells Spring to dispatch the Tiles definition with name `index` whenever a user visits `/` in the web browser.

To wrap it all up we need to define a [deployment descriptor][Descriptor]{:target="_blank"} for our web environment. However we are 
using `Servlet 3.1.0` for this project and there is a programmable alternative introduced in Servlet 3.0 through the
[`javax.servlet.ServletContainerInitializer`](https://docs.oracle.com/javaee/7/api/javax/servlet/ServletContainerInitializer.html){:target="_blank"} interface.  
We will create an implementation of `ServletContainerInitializer` to serve as our deployment descriptor by extending Spring's Abstract
class `AbstractAnnotationConfigDispatcherServletInitializer`.

`src/main/java/com/jipasoft/tiles/initializer/WebInitializer.java`:

{% highlight java %}
public class WebInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

  @Override
  protected Class<?>[] getRootConfigClasses() {
    return null;
  }

  @Override
  protected Class<?>[] getServletConfigClasses() {
    return new Class<?>[] { SpringConfig.class };
  }

  @Override
  protected String[] getServletMappings() {
    return new String[] { "/" };
  }

}
{% endhighlight %}

Run this project and visit `/` in your browser.

# Conclusion
In this post we learned how to integrate the Spring Framework with the Tiles Framework. You can find the source to this guide 
{% include source.html %}. Until the next post, keep doing cool things :+1:.

[Tiles]: https://tiles.apache.org/
[Spring]: https://spring.io/
[Maven]: http://maven.apache.org
[JDK]: http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Descriptor]: https://en.wikipedia.org/wiki/Deployment_descriptor