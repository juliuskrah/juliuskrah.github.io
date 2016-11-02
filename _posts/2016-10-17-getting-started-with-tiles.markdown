---
layout:     post
title:      "Getting Started with Tiles"
date:       2016-10-17 23:00:31 +0000
categories: blog
tags:       java tiles maven
section:    blog
author:     Julius Krah
repo:       apache-tiles-basic-example
---
> [Apache Tiles][Tiles] is a free open-sourced templating framework for modern Java applications. Based upon the Composite pattern 
  it is built to simplify the development of user interfaces.

# Introduction
Building web applications most often than not requires duplicating page elements across several pages. When adding page navigation
for instance, one is required to copy/paste these elements on all pages with navigation. This can sometimes prove to be stressfull
and wrought with errors.  
To avoid code duplication, developers look for tools and/or libraries to make this easier. Shared page elements can be created as
independent homogeneous units and reused across the entire web application using these tools.  
One of such tools that goes a step futher is `Apache Tiles`. `Apache Tiles` is a template composition framework. Tiles was originally
built to simplify the development of web application user interfaces, but it is no longer restricted to the JavaEE web environment. 
For complex web sites it remains the easiest and most elegant way to work alongside any MVC technology. Tiles allows authors to define
page fragments which can be assembled into a complete page at runtime. These fragments, or tiles, can be used as simple includes in
order to reduce the duplication of common page elements or embedded within other tiles to develop a series of reusable templates.
These templates streamline the development of a consistent look and feel across an entire application.  
The final working sample can be found {% include source.html %}. For advanced usage of tiles refer to this [blog post]({% post_url 2016-10-30-tiles-nesting-and-extending-definitions%}).

## Structure
At the end of this guide our folder structure will look similar to this:

```
- ROOT
  - src/
    - main/
      - webapp/
        - layouts/
          - classic.jsp
        - tiles/
          - banner.jsp
          - blog_header.jsp
          - common_menu.jsp
          - credits.jsp
          - home_body.jsp
          - navigation.jsp
        - WEB-INF/
          - tiles.xml
          - web.xml
        - index.jsp
  - pom.xml
```


# Pre-requisites
- [Java Development Kit][JDK]  
- [Maven][]

# Getting dependencies
In this getting started guide we will use `Maven` to manage the dependencies. Let's add the following to our `pom.xml`:

{% highlight xml %}
<dependencies>
  ...
  <dependency>
    <groupId>org.apache.tiles</groupId>
    <artifactId>tiles-extras</artifactId>
    <version>${tiles-version}</version>
  </dependency>
  ...
</dependencies>
{% endhighlight %}

The dependency on `tiles-extras` pulls in all transitive dependencies of tiles. For this guide, we will be using tiles in a web
environment and as such require `servlet-api` and `jsp` jars:

{% highlight xml %}
<dependencies>
  ...
  <dependency>
    <groupId>javax.servlet.jsp.jstl</groupId>
    <artifactId>javax.servlet.jsp.jstl-api</artifactId>
    <version>${jsp.jstl.version}</version>
    <scope>compile</scope>
  </dependency>
  <dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>${servlet.api.version}</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>javax.servlet.jsp</groupId>
    <artifactId>javax.servlet.jsp-api</artifactId>
    <version>${jsp.api.version}</version>
    <scope>provided</scope>
  </dependency>
  ...
</dependencies>
{% endhighlight %}

The versions of the various dependencies are represented as placeholders to ease version management:

{% highlight xml %}
<properties>
  ...
  <tiles-version>3.0.7</tiles-version>
  <servlet.api.version>3.1.0</servlet.api.version>
  <jsp.api.version>2.3.1</jsp.api.version>
  <jsp.jstl.version>1.2.1</jsp.jstl.version>
</properties>
{% endhighlight %}

# Setting up Tiles
`tiles-definitions` are set up in a file called `tiles.xml`. Add the following snippet to your `src/main/webapp/WEB-INF/tiles.xml`
file:

{% highlight xml %}
<tiles-definitions>
  <definition name="myapp.homepage" template="/layouts/classic.jsp">
    <put-attribute name="title" value="Tiles getting started homepage" />
    ...
  </definition>
</tiles-definitions>
{% endhighlight %}

The places of interest to note in the above snippet is the `name` and `template` attributes of the `<definition>` element. The `name`
attribute is used to identify the `tiles-definition` (we would come back to this later) and the `template` attribute to specify 
where the template file can be found relative to the `servlet context` i.e. `src/main/webapp`.  
The child element `<put-attribute>` of parent element `<definition>` specifies a `name/value` pair in our tiles definition. In this
instance the page title.  
Let us now use the definitions in our `jsp` template `(src/main/webapp/layouts/classic.jsp)`:

{% highlight html %}
<jsp:root xmlns:jsp="http://java.sun.com/JSP/Page"
  xmlns:tiles="http://tiles.apache.org/tags-tiles"
  xmlns:c="http://java.sun.com/jsp/jstl/core" 
  version="2.0">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
  <c:url value="/resources/images/favicon.ico" var="faviconurl" />
  <link rel="icon" href="${faviconurl}" />
  <title><tiles:getAsString name="title" /></title>
</head>

<body>
...
</body>

</html>
{% endhighlight %}

Notice the `<tiles>` child element within the `<title>` element. It is tagged with `getAsString` which retrieves the `title` property
`(Tiles getting started homepage)` from the `<put-attribute>`. To run this we need one more configuration bootstrap in `web.xml` 
`(src/main/webapp/WEB-INF/web.xml)`:

> Also take note of the `tiles` namespace on the second line `(xmlns:tiles="http://tiles.apache.org/tags-tiles")`

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns="http://xmlns.jcp.org/xml/ns/javaee"
  xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee 
                      http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
  version="3.1">
  <display-name>apache-tiles-gs</display-name>

  <listener>
    <listener-class>org.apache.tiles.extras.complete.CompleteAutoloadTilesListener</listener-class>
  </listener>

  <servlet>
    <servlet-name>Tiles Dispatch Servlet</servlet-name>
    <servlet-class>org.apache.tiles.web.util.TilesDispatchServlet</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>Tiles Dispatch Servlet</servlet-name>
    <url-pattern>*.tiles</url-pattern>
  </servlet-mapping>

  <welcome-file-list>
    <welcome-file>index.html</welcome-file>
    <welcome-file>index.htm</welcome-file>
    <welcome-file>index.jsp</welcome-file>
    <welcome-file>default.html</welcome-file>
    <welcome-file>default.htm</welcome-file>
    <welcome-file>default.jsp</welcome-file>
  </welcome-file-list>
</web-app>
{% endhighlight %}

> `org.apache.tiles.extras.complete.CompleteAutoloadTilesListener` is used to load the tiles container.  
  For this Guide, we have configured Tiles to work directly with the servlet API, without a controller using 
  `org.apache.tiles.web.util.TilesDispatchServlet`. In the real world, you'll probably use an MVC framework like 
  [Spring][] or [Struts][]. You have to configure your framework to work with Tiles; please refer to your framework's 
  documentation for that.

Run this in any compartible servlet 3.1 container to view the output.

# Adding attributes
Now that we have seen how to create `definitions`, let us add some more `attributes` to this definition `(tiles.xml)`:  

{% highlight xml %}
<tiles-definitions>
  <definition name="myapp.homepage" template="/layouts/classic.jsp">
    ...
    <put-attribute name="header" value="/tiles/banner.jsp" />
    <put-attribute name="menu" value="/tiles/common_menu.jsp" />
    <put-attribute name="body" value="/tiles/home_body.jsp" />
    <put-attribute name="footer" value="/tiles/credits.jsp" />
    <put-attribute name="heading" value="/tiles/blog_header.jsp" />
    <put-attribute name="navigation" value="/tiles/navigation.jsp" />
  </definition>
</tiles-definitions>
{% endhighlight %}

The `<put-attribute>` elements above all reference file fragments in the `src/main/webapp/tiles/` directory. We would create this 
directory and add the following files to it:

- `banner.jsp`
- `blog_header.jsp`
- `common_menu.jsp`
- `credits.jsp`
- `home_body.jsp`
- `navigation.jsp`

Let us add some content to our `JSP` files.  
`src/main/webapp/tiles/banner.jsp`:

{% highlight html %}
<div>
  <nav>
    <a href="#">Home</a> 
    <a href="#">New features</a> 
    <a href="#">Press</a> 
    <a href="#">New hires</a> 
    <a href="#">About</a>
  </nav>
</div>
{% endhighlight %}


`src/main/webapp/tiles/blog_header.jsp`:

{% highlight html %}
<h1 class="blog-title">Getting started with Tiles</h1>
<p class="lead blog-description">
  The official example template of creating a blog with Bootstrap.
</p>
{% endhighlight %}

`src/main/webapp/tiles/common_menu.jsp`:

{% highlight html %}
<div>
  <h4>About</h4>
  <p>
    Etiam porta <em>sem malesuada magna</em> mollis euismod. Cras mattis
    consectetur purus sit amet fermentum. Aenean lacinia bibendum nulla
    sed consectetur.
  </p>
</div>
<div>
  <h4>Archives</h4>
  <ol>
    <li><a href="#">March 2014</a></li>
    <li><a href="#">February 2014</a></li>
    <li><a href="#">January 2014</a></li>
    <li><a href="#">December 2013</a></li>
    <li><a href="#">November 2013</a></li>
    <li><a href="#">October 2013</a></li>
    <li><a href="#">September 2013</a></li>
    <li><a href="#">August 2013</a></li>
    <li><a href="#">July 2013</a></li>
    <li><a href="#">June 2013</a></li>
    <li><a href="#">May 2013</a></li>
    <li><a href="#">April 2013</a></li>
  </ol>
</div>
<div>
  <h4>Elsewhere</h4>
  <ol>
    <li><a href="#">GitHub</a></li>
    <li><a href="#">Twitter</a></li>
    <li><a href="#">Facebook</a></li>
  </ol>
</div>
{% endhighlight %}

`src/main/webapp/tiles/credits.jsp`, our example project makes use of [Twitter Bootstrap][Bootstrap]:

{% highlight html %}
<p>
  Blog template built for 
  <a href="//getbootstrap.com">Bootstrap</a> by 
  <a href="//twitter.com/mdo">@mdo</a>.
</p>
<p>
  <a href="#">Back to top</a>
</p>
{% endhighlight %}

`src/main/webapp/tiles/home_body.jsp`:

{% highlight html %}
<div>
  <h2>Sample blog post</h2>
  <p>
    January 1, 2014 by <a href="#">Mark</a>
  </p>

  <p>
    This blog post shows a few different types of content that's
    supported and styled with Bootstrap. Basic typography, images, and
    code are all supported.
  </p>
  <hr/>
  <p>
    Cum sociis natoque penatibus et magnis 
    <a href="#">dis parturient montes</a>, 
    nascetur ridiculus mus. Aenean eu leo quam. Pellentesque
    ornare sem lacinia quam venenatis vestibulum. Sed posuere consectetur
    est at lobortis. Cras mattis consectetur purus sit amet fermentum.
  </p>
  <blockquote>
    <p>
      Curabitur blandit tempus porttitor. 
      <strong>Nullam quis risus eget urna mollis</strong> 
      ornare vel eu leo. Nullam id dolor id nibh
      ultricies vehicula ut id elit.
    </p>
  </blockquote>
  <p>
    Etiam porta 
    <em>sem malesuada magna</em> mollis euismod. Cras mattis
    consectetur purus sit amet fermentum. Aenean lacinia bibendum nulla
    sed consectetur.
  </p>
</div>
{% endhighlight %}

`src/main/webapp/tiles/navigation.jsp`

{% highlight html %}
<nav>
  <ul>
    <li><a href="#">Previous</a></li>
    <li><a href="#">Next</a></li>
  </ul>
</nav>
{% endhighlight %}

# Putting it all together
To wrap up, this is the complete `tiles-definition`:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE tiles-definitions PUBLIC
  "-//Apache Software Foundation//DTD Tiles Configuration 3.0//EN"
  "http://tiles.apache.org/dtds/tiles-config_3_0.dtd">
<tiles-definitions>
  <definition name="myapp.homepage" template="/layouts/classic.jsp">
    <put-attribute name="title" value="Tiles tutorial homepage" />
    <put-attribute name="header" value="/tiles/banner.jsp" />
    <put-attribute name="menu" value="/tiles/common_menu.jsp" />
    <put-attribute name="body" value="/tiles/home_body.jsp" />
    <put-attribute name="footer" value="/tiles/credits.jsp" />
    <put-attribute name="heading" value="/tiles/blog_header.jsp" />
    <put-attribute name="navigation" value="/tiles/navigation.jsp" />
  </definition>
</tiles-definitions>
{% endhighlight %}

Now in our welcome page `(src/main/webapp/index.jsp)` let us make use of our tiles definition name (Remember I said we would 
come back to it) `myapp.homepage`:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8" ?>
<jsp:root xmlns:jsp="http://java.sun.com/JSP/Page"
  xmlns:tiles="http://tiles.apache.org/tags-tiles"
  xmlns:c="http://java.sun.com/jsp/jstl/core" version="2.0">
  <jsp:directive.page contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8" session="false" />
  <jsp:output doctype-root-element="html"
    doctype-public="-//W3C//DTD XHTML 1.0 Transitional//EN"
    doctype-system="http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd"
    omit-xml-declaration="true" />

  <tiles:insertDefinition name="myapp.homepage" />

</jsp:root>
{% endhighlight %}

Notice the `<tiles>` element right before the `<jsp:root>` closing tag, we are calling the tiles-definition by name. This tells
tiles to insert the `classic.jsp` layout in the index page.  
Run this in any servlet 3.1 container to see the final output.

> When tiles dispatcher servlet is configured, you can serve definition names directly with *.tiles e.g. `/myapp.homepage.tiles`:

`src/main/webapp/WEB-INF/web.xml`
{% highlight xml %}
...
<servlet>
  <servlet-name>Tiles Dispatch Servlet</servlet-name>
  <servlet-class>org.apache.tiles.web.util.TilesDispatchServlet</servlet-class>
</servlet>
<servlet-mapping>
  <servlet-name>Tiles Dispatch Servlet</servlet-name>
  <url-pattern>*.tiles</url-pattern>
</servlet-mapping>
...
{% endhighlight %}

# Conclusion
Tiles can be used with web frameworks such as [Spring][] and [Struts][], and template technologies like JSP and [Thymeleaf][].  
You can find the source to this guide {% include source.html %}. Happy coding :+1:.



[Maven]: http://maven.apache.org
[Tiles]: https://tiles.apache.org/framework/index.html
[JDK]: http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Struts]: http://struts.apache.org/
[Spring]: https://spring.io
[Bootstrap]: http://getbootstrap.com/
[Thymeleaf]: http://www.thymeleaf.org/
