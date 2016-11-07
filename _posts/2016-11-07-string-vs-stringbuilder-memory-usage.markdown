---
layout:     post
title:      'String vs StringBuilder Memory Usage'
date:       2016-11-07 16:35:45 +0000
categories: gist
tags:       java spring-boot maven
section:    blog
author:     Julius Krah
---
When writing an application in java, there are several things to look out for. What is the optimization consideration? Speed 
or Memory? This begs the question, when do you use `String` and `StringBuilder`?  
When using StringBuilder you're reserving space to add strings to your buffer. When you're using a String, every time you 
type something like this:

{% highlight java %}
String s = s + someOtherString;
{% endhighlight %}

You're throwing away your existing `s` string and creating a new one to go instead of it which is `s + someOtherString`. 
This means that you're constantly needing new memory to make the concatenated string, throw the old one out, and put the 
reference of your variable to your new string.

With a `StringBuilder` you can append to your string without having to remove the existing part.

So yes, it uses more memory, but it's very efficient in some scenario's compared to using just a string.

In other words: String is an immutable object and a StringBuilder isn't.

Let us demonstrated with an example: 

{% gist cb7f72036de45683a432e39e85333367 Application.java %}

It is a maven project running with `Spring-Boot`:

{% gist cb7f72036de45683a432e39e85333367 pom.xml %}

Cheers :smile:.