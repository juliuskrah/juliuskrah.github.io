---
layout:     post
title:      'Tiles: Nesting and Extending Definitions'
date:       2016-10-30 22:35:31 +0000
categories: blog
tags:       java tiles maven
section:    blog
author:     juliuskrah
repo:       apache-tiles-advanced-example
---
> [Apache Tiles][Tiles]{:target="_blank"} is a free open-sourced templating framework for modern Java applications. Based upon the Composite pattern 
  it is built to simplify the development of user interfaces.

# Introduction
In this blog post we are going to learn how to `nest` and `extend` tiles definitions. If you are new to `tiles` I recommend you read
this [blog post]({% post_url 2016-10-17-getting-started-with-tiles %}) first. Sometimes it is useful to have a structured page with, 
say, a structured body. Typically, there is a main layout (for example, the "classic" layout) and the body is made of certain number 
of sections. In this case, nesting a definition (the one for the body) inside another definition (the main layout) can be useful.  
You can follow along this guide or get the final example {% include source.html %}.

# Prerequisites
- [Java Development Kit][JDK]{:target="_blank"}  
- [Maven][]{:target="_blank"}

# Getting the Dependencies
In this guide we will use `Maven` to manage the dependencies. Let's add the following to our `pom.xml`:

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

The versions of the various dependencies are given as properties to ease version management:

{% highlight xml %}
<properties>
  ...
  <tiles-version>3.0.7</tiles-version>
  <servlet.api.version>3.1.0</servlet.api.version>
  <jsp.api.version>2.3.1</jsp.api.version>
  <jsp.jstl.version>1.2.1</jsp.jstl.version>
</properties>
{% endhighlight %}

# Nesting Definitions
Sometimes it is useful to have a structured page with, say, a structured body. Typically, there is a main layout (for example, 
the "classic" layout) and the body is made of certain number of sections. In this case, nesting a definition (the one for the 
body) inside another definition (the main layout) can be useful.

## Named Subdefinitions
Tiles supports nesting definitions natively. One way of using nested definitions is creating a named "subdefinition" and using 
it as an attribute. For example:

{% highlight xml %}
<definition name="myapp.homepage.body" template="/layouts/home_body.jsp">
  <put-attribute name="one" value="/tiles/blog_one.jsp" />
  <put-attribute name="two" value="/tiles/blog_two.jsp" />
  <put-attribute name="three" value="/tiles/blog_three.jsp" />
</definition>

<definition name="myapp.homepage" template="/layouts/classic.jsp">
  <put-attribute name="title" value="Tiles: Nesting and Extending Definitions" />
  <put-attribute name="header" value="/tiles/banner.jsp" />
  <put-attribute name="menu" value="/tiles/common_menu.jsp" />
  <put-attribute name="body" value="myapp.homepage.body" />
  <put-attribute name="footer" value="/tiles/credits.jsp" />
  <put-attribute name="heading" value="/tiles/blog_header.jsp" />
  <put-attribute name="navigation" value="/tiles/navigation.jsp" />
</definition>
{% endhighlight %}

In the above snippet we have nested the `myapp.homepage.body` definition inside `myapp.homepage`, by putting it 
inside its `body` attribute. We will be seeing the definition one inside the other.

## Anonymous Nested Definitions
What you can do with named subdefinitions can be done with nested anonymous definitions. The above example can be rewritten in:

{% highlight xml %}
<definition name="myapp.homepage" template="/layouts/classic.jsp">
  <put-attribute name="title" value="Tiles: Nesting and Extending Definitions" />
  <put-attribute name="header" value="/tiles/banner.jsp" />
  <put-attribute name="menu" value="/tiles/common_menu.jsp" />
  <put-attribute name="body">
    <definition template="/layouts/home_body.jsp">
      <put-attribute name="one" value="/tiles/blog_one.jsp" />
      <put-attribute name="two" value="/tiles/blog_two.jsp" />
      <put-attribute name="three" value="/tiles/blog_three.jsp" />
    </definition>
  </put-attribute>
  <put-attribute name="footer" value="/tiles/credits.jsp" />
  <put-attribute name="heading" value="/tiles/blog_header.jsp" />
  <put-attribute name="navigation" value="/tiles/navigation.jsp" />
</definition>
{% endhighlight %}

The anonymous definition put under the `body` attribute can be used only by the surrounding definition. Moreover, you can nest a 
definition into a nested definition, with the desired level of depth.

## Cascaded Attributes
Attributes defined into a definition can be cascaded to be available to all nested definitions and templates. For example the 
definition detailed above can be rewritten this way:

{% highlight xml %}
<definition name="myapp.homepage" template="/layouts/classic.jsp">
  <put-attribute name="title" value="Tiles: Nesting and Extending Definitions" />
  <put-attribute name="header" value="/tiles/banner.jsp" />
  <put-attribute name="menu" value="/tiles/common_menu.jsp" />
  <put-attribute name="body" value="/layouts/home_body.jsp" />
  <put-attribute name="footer" value="/tiles/credits.jsp" />
  <put-attribute name="heading" value="/tiles/blog_header.jsp" />
  <put-attribute name="navigation" value="/tiles/navigation.jsp" />

  <put-attribute name="one" value="/tiles/blog_one.jsp" cascade="true" />
  <put-attribute name="two" value="/tiles/blog_two.jsp" cascade="true" />
  <put-attribute name="three" value="/tiles/blog_three.jsp" cascade="true" />
</definition>
{% endhighlight %}

The template of `myapp.homepage.body` definition has been used as the `body` attribute in the `myapp.homepage` definition. 
All of the attributes of `myapp.homepage.body` has been then moved as attributes of `myapp.homepage` definition, but with 
the addition of the "cascade" flag.

Our `/layouts/home_body.jsp` will contain this content:

{% highlight html %}
<jsp:root xmlns:jsp="http://java.sun.com/JSP/Page"
  xmlns:tiles="http://tiles.apache.org/tags-tiles" version="2.0">

  <tiles:insertAttribute name="one" />
  <tiles:insertAttribute name="two" />
  <tiles:insertAttribute name="three" />

</jsp:root>
{% endhighlight %}

# Extending Definitions
You can extend definitions like a Java class. The concepts of `abstract` definition, `extension` and `override` are available.

## Abstract Definition
It is a definition in which the template attributes are not completely filled. They are useful to create a base page and a 
number of extending definitions, reusing already created layout. For example:

{% highlight xml %}
<definition name="myapp.homepage" template="/layouts/classic.jsp">
  <put-attribute name="header" value="/tiles/banner.jsp" />
  <put-attribute name="menu" value="/tiles/common_menu.jsp" />
  <put-attribute name="body">
    <definition template="/layouts/home_body.jsp">
      <put-attribute name="one" value="/tiles/blog_one.jsp" />
      <put-attribute name="two" value="/tiles/blog_two.jsp" />
      <put-attribute name="three" value="/tiles/blog_three.jsp" />
    </definition>
  </put-attribute>
  <put-attribute name="footer" value="/tiles/credits.jsp" />
  <put-attribute name="navigation" value="/tiles/navigation.jsp" />
</definition>
{% endhighlight %}

## Definition Extension
A definition can inherit from another definition, to reuse an already made (abstract or not) definition:

{% highlight xml %}
<definition name="myapp.new-features" extends="myapp.homepage">
  <put-attribute name="title" value="Extended Definition" />
  <put-attribute name="heading" value="/tiles/new_features_header.jsp" />
</definition>
{% endhighlight %}

In this case, the `header`, `menu`, `body`, `footer` and `navigation` are inherited from the `myapp.page` definition, while the rest 
is defined inside the "concrete" definition.

## Template and Attribute Override
When extending a definition, its template and attributes can be overridden.

*Overriding a Template*

{% highlight xml %}
<definition name="myapp.list" template="/layouts/variable_rows.jsp" extends="myapp.homepage">
{% endhighlight %}

The definition has the same attributes, but its template changed. The result is that the content is the same, 
but the layout is different.

*Overriding Attributes*

{% highlight xml %}
<definition name="myapp.new-features" extends="myapp.homepage">
  <put-attribute name="heading" value="/tiles/new_features_header.jsp" />
</definition>
{% endhighlight %}

In this case, the page will have the same appearance as the `myapp.homepage` definition, but its `heading` subpage is different.

# Putting it all Together
We will first create our welcome page in the `layouts` folder `(/layouts/classic.jsp)`:

{% highlight html %}
<?xml version="1.0" encoding="UTF-8" ?>
<jsp:root xmlns:jsp="http://java.sun.com/JSP/Page"
  xmlns:tiles="http://tiles.apache.org/tags-tiles" version="2.0">
  <jsp:directive.page contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8" session="false" />
  <jsp:output doctype-root-element="html"
    doctype-public="-//W3C//DTD XHTML 1.0 Transitional//EN"
    doctype-system="http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd"
    omit-xml-declaration="true" />
<html>
<head>
  <title><tiles:getAsString name="title" /></title>
</head>
<body>
  <div>
    <tiles:insertAttribute name="header" />
  </div>
  <div>
    <div>
      <tiles:insertAttribute name="heading" />
    </div>
    <div>
      <div>
        <tiles:insertAttribute name="body" />
        <tiles:insertAttribute name="navigation" />
      </div>
      <div>
        <tiles:insertAttribute name="menu" />
      </div>
    </div>
  </div>
  <footer>
    <tiles:insertAttribute name="footer" />
  </footer>
</body>
</html>
</jsp:root>
{% endhighlight %}

Our next jsp template is the `banner.jsp` in the `tiles` folder `(/tiles/banner.jsp)`:

{% highlight html %}
<jsp:root xmlns:jsp="http://java.sun.com/JSP/Page"
  xmlns:c="http://java.sun.com/jsp/jstl/core" version="2.0">
  <c:url value="/myapp.new-features.tiles" var="newfeatures" />
  <c:url value="/myapp.homepage.tiles" var="homepage" />
  <c:url value="/myapp.list.tiles" var="list" />
  <div>
    <nav>
      <a href="${homepage}">Home</a> 
      <a href="${newfeatures}">Override</a> 
      <a href="${list}">List</a> 
      <a href="#">Dummy</a> 
      <a href="#">About</a>
    </nav>
  </div>
</jsp:root>
{% endhighlight %}

`/tiles/blog_header.jsp`:

{% highlight html %}
<h1 class="blog-title">The Bootstrap Blog</h1>
<p class="lead blog-description">The official example template of creating a blog with Bootstrap.</p>
{% endhighlight %}

# Conclusion
In this blog post, we have looked at how to extend and nest tiles definition. As usual you can find the source to this guide 
{% include source.html %}. Until the next post, keep doing cool things :+1:.





[Maven]: http://maven.apache.org
[Tiles]: https://tiles.apache.org/framework/index.html
[JDK]: http://www.oracle.com/technetwork/java/javase/downloads/index.html