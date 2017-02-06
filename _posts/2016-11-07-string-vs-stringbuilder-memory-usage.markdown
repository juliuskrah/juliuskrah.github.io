---
layout:     post
title:      'String vs StringBuilder Memory Usage'
date:       2016-11-07 16:35:45 +0000
categories: gist
tags:       java spring-boot maven
section:    blog
author:     juliuskrah
---
When writing an application in Java, there are several things to look out for. What is the optimization consideration? Speed 
or Memory? This begs the question, when do you use `String` and `StringBuilder`?  
When using `StringBuilder` you're reserving space to add strings to your buffer. When you're using a `String`, every time you 
type something like this:

{% highlight java %}
String s = s + someOtherString;
{% endhighlight %}

you're throwing away your existing `s` string and creating a new one.  
This means that you're constantly needing new memory to make the concatenated string, throw the old one out, and put the 
reference of your variable to your new string. To put this in perspective refer to this [SO question](http://stackoverflow.com/questions/2721998/how-java-do-the-string-concatenation-using){:target="_blank"}.

With a `StringBuilder` you can append to your string without having to remove the existing part.

Consider this example: 

{% gist cb7f72036de45683a432e39e85333367 Application.java %}

Replace `<Large File>` on lines 50 and 69 with the path to a large file and run the sample. After running this snippet, you would
notice the `String` method consumed more memory. On the other hand `StringBuilder` finished processing faster.

The `String` class represents character strings. All string literals in Java programs, such as `"abc"`, are implemented as 
instances of this class.  
`String`s are constant (immutable); their values cannot be changed after they are created. String buffers support mutable 
strings. Because String objects are immutable they can be shared.  
The Java language provides special support for the string concatenation operator ( + ), and for conversion of other objects to 
strings. `String` concatenation is implemented through the `StringBuilder`(or `StringBuffer`) class and its append method. `String` 
conversions are implemented through the method toString, defined by `Object` and inherited by all classes in Java.

`StringBuilder` objects are like `String` objects, except that they can be modified (mutable). Internally, these objects 
are treated like variable-length arrays that contain a sequence of characters. At any point, the length and content of 
the sequence can be changed through method invocations.  
`String`s should always be used unless string builders offer an advantage in terms of simpler code or better performance. 
For example, if you need to concatenate a large number of strings, appending to a StringBuilder object is more efficient.  
The `StringBuilder` class, like the `String` class, has a `length()` method that returns the length of the character sequence
in the builder.  
Unlike strings, every string builder also has a capacity, the number of character spaces that have been allocated. The 
capacity, which is returned by the `capacity()` method, is always greater than or equal to the length (usually greater than) 
and will automatically expand as necessary to accommodate additions to the string builder.

 Maven is used to manage the dependencies of this `Spring-Boot` sample:

{% gist cb7f72036de45683a432e39e85333367 pom.xml %}

Back to the question raised at the beginning of this post. Which would you consider? It depends. When you are writing reactively
fast applications and speed is a concern, you may consider `StringBuilder`. On the other hand when memory is a major constraint and
GC overhead is an issue then `String` may very well suit your needs. This does not however lock you to use one or the other. Feel 
free using both in the same application wherever they are best suited. Until the next post, keep doing cool things :smile:.