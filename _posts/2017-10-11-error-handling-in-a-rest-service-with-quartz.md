---
layout:     series
title:      Error Handling in a REST Service with Quartz
date:       2017-10-11 22:15:46 +0000
categories: tutorial
tags:       java maven liquibase quartz rest spring-boot
section:    series
author:     juliuskrah
repo:       quartz-manager/tree/v2.x
---
> Spring MVC provides several complimentary approaches to exception handling.

# Introduction
I welcome you all to this third post in the series of integrating Quartz with Spring.
Througout the course of this tutorial on Quartz, you will notice we have not done any error handling.
In this post I will show you how to configure a Spring REST application to handle errors.

This post builds upon the [previous post]({% post_url 2017-10-06-persisting-dynamic-jobs-with-quartz-and-spring %})
and you can get the source/base for this post as [zip](https://github.com/juliuskrah/quartz-manager/archive/v2.0.zip)|[tar.gz](https://github.com/juliuskrah/quartz-manager/archive/v2.0.tar.gz). Extract the contents of the archive and
let us begin.

# Directory structure
The contents of the archive should be similar to the directory structure below:

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__juliuskrah/
|  |  |  |  |  |__quartz/
|  |  |  |  |  |  |__Application.java
|  |  |  |  |  |  |__autoconfigure/
|  |  |  |  |  |  |  |__QuartzProperties.java
|  |  |  |  |  |  |__job/
|  |  |  |  |  |  |  |__EmailJob.java
|  |  |  |  |  |  |__mail/
|  |  |  |  |  |  |  |__javamail/
|  |  |  |  |  |  |  |  |__AsyncMailSender.java
|  |  |  |  |  |  |__model/
|  |  |  |  |  |  |  |__JobDescriptor.java
|  |  |  |  |  |  |  |__TriggerDescriptor.java
|  |  |  |  |  |  |__service/
|  |  |  |  |  |  |  |__EmailService.java
|  |  |  |  |  |  |__web/
|  |  |  |  |  |  |  |__rest/
|  |  |  |  |  |  |  |  |__EmailResource.java
|  |  |  |  |  |  |  |  |__errors/
|  |  |  |  |  |  |  |  |  |__ErrorVO.java
|  |  |  |  |  |  |  |  |  |__ExceptionTranslator.java
|  |  |  |  |  |  |  |  |  |__FieldErrorVO.java
|  |  |  |__org/
|  |  |  |  |__springframework/
|  |  |  |  |  |__boot/ 
|  |  |  |  |  |  |__autoconfigure/  
|  |  |  |  |  |  |  |__quartz/
|  |  |  |  |  |  |  |  |__AutowireCapableBeanJobFactory.java
|  |  |  |__io/
|  |  |  |  |__github/
|  |  |  |  |  |__jhipster/
|  |  |  |  |  |  |__async/
|  |  |  |  |  |  |  |__ExceptionHandlingAsyncTaskExecutor.java  
|  |  |__resources/
|  |  |  |__db/
|  |  |  |  |__changelog/
|  |  |  |  |  |__db.changelog-master.yaml
|  |  |  |__application.yaml
|  |  |  |__quartz.properties
|__pom.xml
```

## Prerequisites
To follow along with this guide, you should have the following set up on your development machine:

- [Java Development Kit][JDK]{:target="_blank"}

_Optional_

- [Maven][]{:target="_blank"}
- [cURL][]{:target="_blank"}

## Adding Validation
I will start with basic validation by annotating the `*Descriptor` classes with Hibernate Validator annotations.
Hibernate Validator is the Reference Implementation of Bean Validation [JSR 349] (for those of you who like to know):

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/model/TriggerDescriptor.java' %}

{% highlight java %}
public class TriggerDescriptor {
  @NotBlank
  private String name;
  @NotBlank
  private String group;
  @Valid
  private List<TriggerDescriptor> triggerDescriptors = new ArrayList<>();
  ...
}
{% endhighlight %}

The trigger `name` and `group` may not be empty or null.

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/model/JobDescriptor.java' %}

{% highlight java %}
public class JobDescriptor {
  @NotBlank
  private String name;
  private String group;
  @NotEmpty
  private String subject;
  @NotEmpty
  private String messageBody;
  @NotEmpty
  private List<String> to;
  ...
}
{% endhighlight %}

As you can see from the above, there is no validation on the `group`. This is because the API consumers are not 
required to specify a group name in the request payload but rather in the URL as a `PathVariable`.

To activate the validation, annotate the `JobDescriptor` parameter with `@Valid`:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/web/rest/EmailResource.java' %}

{% highlight java %}
public class EmailResource {
  @PostMapping(path = "/groups/{group}/jobs")
  public ResponseEntity<JobDescriptor> createJob(@PathVariable String group, 
            @Valid @RequestBody JobDescriptor descriptor) {
    //
  }
  
  @PutMapping(path = "/groups/{group}/jobs/{name}")
  public ResponseEntity<Void> updateJob(@PathVariable String group, 
              @PathVariable String name, 
              @Valid @RequestBody JobDescriptor descriptor) {
    //
  }
}
{% endhighlight %}

You can test this out by making a `POST` or `PUT` request and leave the `name` field blank.

## Adding Exceptions
Let us start with throwing `Unchecked` exceptions for a few known exception prone areas. When an API 
consumer creates a job and specifies its trigger(s) and fails to add either `cron` or `fireTime` in the
trigger(s) we will throw an `IllegalStateException`:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/model/TriggerDescriptor.java' %}

{% highlight java %}
public class TriggerDescriptor {
  ...
  public Trigger buildTrigger() {
    if (!isEmpty(cron)) {
      //
    } else if (!isEmpty(fireTime)) {
      //
    }
    throw new IllegalStateException("unsupported trigger descriptor " + this);
  }
}
{% endhighlight %}

We can also validate the given `cron` expression and throw `IllegalArgumentException` if expression is invalid:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/model/TriggerDescriptor.java' %}

{% highlight java %}
public class TriggerDescriptor {
  ...
  public Trigger buildTrigger() {
    if (!isEmpty(cron)) {
      if (!isValidExpression(cron))
        throw new IllegalArgumentException("Provided expression '" + cron + "' is not a valid cron expression");
      //
    }
  }
}
{% endhighlight %}

We need to ensure that consumers of our API do not register jobs with an existing `name` and `group` in the 
scheduler:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/service/EmailService.java' %}

{% highlight java %}
public class EmailService extends AbstractJobService {
  ...
  public JobDescriptor createJob(String group, JobDescriptor descriptor) {
    String name = ...;
    try {
      if (scheduler.checkExists(jobKey(name, group)))
        throw new DataIntegrityViolationException("Job with Key '" + group + "." + name +"' already exists");
      ...
    } catch (SchedulerException e) {
      //
    }
    return descriptor;
  }
}
{% endhighlight %}

I will show you how to translate these exceptions into meaningful `4xx` status codes to the API consumer in the 
next section.

## Adding Exception Translators
We will create two immutable value objects (`FieldErrorVO` and `ErrorVO`) to transfer the errors to the API consumers.
To ease this creation we will use [Immutables][]{:target="_blank"}:

file: {% include file-path.html file_path='pom.xml' %}

{% highlight xml %}
...
<dependency>
  <groupId>org.immutables</groupId>
  <artifactId>value</artifactId>
  <version>2.5.6</version>
  <scope>provided</scope>
</dependency>
{% endhighlight %}

Immutables is an annotation processor and if you are using an IDE you can activate it by following the instructions
laid out in this [link](http://immutables.github.io/apt.html){:target="_blank"}.

We will create a value object to map field validation errors:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/web/rest/errors/FieldErrorVO.java' %}

{% highlight java %}
@Value.Immutable
@JsonSerialize(as = ImmutableFieldErrorVO.class)
@JsonDeserialize(as = ImmutableFieldErrorVO.class)
public interface FieldErrorVO {
  String objectName();
  String field();
  String message();
}
{% endhighlight %}

> While `ImmutableFieldErrorVO` may not be generated yet, the above will compile properly

And another value object to map all other errors:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/web/rest/errors/ErrorVO.java' %}

{% highlight java %}
@Value.Immutable
@JsonSerialize(as = ImmutableErrorVO.class)
@JsonDeserialize(as = ImmutableErrorVO.class)
public interface ErrorVO {
  String message();
  String description();
  List<FieldErrorVO> fieldErrors();
}
{% endhighlight %}

> While `ImmutableErrorVO` may not be generated yet, the above will compile properly

Now we create the `ExceptionTranslator` class:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/web/rest/errors/ExceptionTranslator.java' %}

{% highlight java %}
@RestControllerAdvice
public class ExceptionTranslator {
  @ExceptionHandler(IllegalStateException.class)
  @ResponseStatus(BAD_REQUEST)
  public ErrorVO processUnsupportedTriggerError(IllegalStateException ex) {
    ErrorVO dto = ImmutableErrorVO.builder()
        .message("400: Bad Request")
        .description(ex.getMessage())
        .build();
    return dto;
  }
	
  @ExceptionHandler(IllegalArgumentException.class)
  @ResponseStatus(BAD_REQUEST)
  public ErrorVO processInvalidCronExpressionError(IllegalArgumentException ex) {
    //
  }
	
  @ExceptionHandler(MethodArgumentNotValidException.class)
  @ResponseStatus(BAD_REQUEST)
  public ErrorVO processValidationError(MethodArgumentNotValidException ex) {
    ...
    List<FieldError> fieldErrors = ...;
    ImmutableErrorVO.Builder builder = ...;
        
    for (FieldError fieldError : fieldErrors) {
      builder.addFieldErrors(ImmutableFieldErrorVO.builder()
        .objectName(fieldError.getObjectName())
        .field(fieldError.getField())
        .message(fieldError.getCode())
        .build());
    }
    return builder.build();
  }
	
  @ExceptionHandler(DataIntegrityViolationException.class)
  @ResponseStatus(CONFLICT)
  public ErrorVO processDataIntegrityViolationError(DataIntegrityViolationException ex) {
    //
  }
	
  @ExceptionHandler(Exception.class)
  public ResponseEntity<ErrorVO> processException(Exception ex) throws Exception {
    //
  }
}
{% endhighlight %}

That's all folks

# Conclusion
In this post we learned how to validate and handle errors from the client and server. You can extend this to create
powerful and robust exception handling for your APIs.

You can find the source to this guide {% include source.html %}. Until the next post, keep doing cool things :smile:.

[cURL]:                                       https://curl.haxx.se/
[Immutables]:                                 http://immutables.github.io/
[JDK]:                                        http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                                      http://maven.apache.org