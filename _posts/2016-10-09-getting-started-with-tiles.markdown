---
layout:     post
title:      "Getting Started with Tiles"
date:       2016-10-09 22:05:31 +0000
categories: blog java tiles
section:    blog
---
> Brief description of `Apache Tiles` goes here.

# Introduction
Building web applications most often than not requires duplicating page elements across several pages. When adding page navigation
for instance, one is required to copy/paste these elements on all pages with navigation. This can sometimes prove to be stressfull
and wrought with errors.  
To avoid code duplication, developers look for tools and/or libraries to make this easier. Shared page elements can be created as
independent homogeneous units and reused across the entire web application using these tools.  
One of such tools that goes a step futher is `Apache Tiles`. `Apache Tiles` is a ...

## Structure
At the end of this guide our folder structure will look similar to this:

```
ROOT\
  |\
  src\
    |\
    main\
      webapp\
        layouts\
          |\
          classic.jsp
        resources\
          |\
          css\
            |\
            blog.css
            bootstrap.min.css
            ie10-viewport-bug-workaround.css
          images\
            |\
            favicon.ico
          js\
            |\
            bootstrap.min.js
            ie-emulation-modes-warning.js
            ie10-viewport-bug-workaround.js
            ie8-responsive-file-warning.js
            jquery.min.js
        tiles\
          |\
          banner.jsp
          blog_header.jsp
          common_menu.jsp
          credits.jsp
          home_body.jsp
          navigation.jsp
        WEB-INF\
          |\
          tiles.xml
          web.xml
        index.jsp
  pom.xml
```


# Pre-requisites
- Java Development Kit  
- [Maven][]

# Getting dependencies
This getting started guide will use `Maven` to manage the dependencies. Add the following to your `pom.xml`:

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
environment and as such require `servlet` and `jsp`:

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
  <tiles-version>3.0.5</tiles-version>
  <servlet.api.version>3.1.0</servlet.api.version>
  <jsp.api.version>2.3.1</jsp.api.version>
  <jsp.jstl.version>1.2.1</jsp.jstl.version>
</properties>
{% endhighlight %}

# Setting up Tiles
Tiles `definitions` are set up in a file called `src/main/webapp/WEB-INF/tiles.xml`. Add the following snippet to your `tiles.xml`
file:

{% highlight xml %}
<tiles-definitions>
  <definition name="myapp.homepage" template="/layouts/classic.jsp">
    <put-attribute name="title" value="Tiles getting started homepage" />
    ...
  </definition>
</tiles-definitions>
{% endhighlight %}

The places of interest to note in the above snippet is the `name` and `template` attributes of the `definition` element. The `name`
attribute is used to identify the `tiles-definition` (we would come back to this later) and the `template` attribute to specify 
where the template file can be found relative to the `servlet context` i.e. `src/main/webapp`.  
The child element `put-attribute` of parent element `definition` specifies a `name/value` pair in our tiles definition. In this
instance the page title.  
Let us now use the definitions in our `jsp` template (`src/main/webapp/layouts/classic.jsp`):

{% highlight html %}
<jsp:root xmlns:jsp="http://java.sun.com/JSP/Page"
  xmlns:tiles="http://tiles.apache.org/tags-tiles"
  xmlns:c="http://java.sun.com/jsp/jstl/core" version="2.0">

<head>
  <c:url value="/resources/images/favicon.ico" var="faviconurl" />
  <link rel="icon" href="${faviconurl}" />
  <title><tiles:getAsString name="title" /></title>
</head>
{% endhighlight %}

Notice the `tiles` child element within the `title` element. It is tagged with `getAsString` which retrieves the `title attribute`
(`Tiles getting started homepage`) from the `tiles definition`. Run this in any compartible servlet 3.1 container to view the 
output.

# Adding attributes


[Maven]: http://maven.apache.org
