---
layout:     post
title:      Task Execution and Scheduling with Spring
date:       2017-06-03 00:19:57 +0000
categories: blog
tags:       java maven spring-boot spring
section:    blog
author:     juliuskrah
repo:       spring-schedular-example/tree/spring-task-executor
---
> The Spring Framework provides abstractions for asynchronous execution and scheduling of tasks with the `TaskExecutor`
  and `TaskScheduler` interfaces, respectively.  
  
# Introduction
In this post we will learn how to schedule jobs and perform asynchronous task execution with Spring.

## How to Complete this Guide
[Download](https://github.com/juliuskrah/spring-schedular-example/archive/initial.zip) and unzip the source repository 
for this guide, or clone it using Git: `git clone https://github.com/juliuskrah/spring-schedular-example.git`

## Prerequisites
- [Java Development Kit][JDK]{:target="_blank"}
- [Maven][]{:target="_blank"}

## Project Structure
This is a `maven` based project and we are using the standard Java project structure.  

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |juliuskrah/  
|  |  |  |  |__Application.java
|  |  |  |  |__execute/
|  |  |  |  |  |__Execute.java
|  |  |  |  |__schedule/
|  |  |  |  |  |__Schedule.java
|  |  |__resources/
|  |  |  |__application.properties
|__pom.xml
```
## Project Dependencies
This is a Spring-Boot based project and we are going to add the following dependencies to our `pom.xml`:

{% highlight xml%}
...
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
</dependency>
{% endhighlight %}

# Task Sheduler
Sometimes you will want to perform an unsupervised task at a future time. For example you have a social media site 
and will want to remove all users who have been inactive for three months. You want this job to be performed at 
midnight each day.  
To illustrate this, we will build a simple scheduled task that removes all users from an embedded `h2` database
with a status of `false`.

{% highlight java %}
@Scheduled(cron = "30 0/1 * * * ?")
public void scheduledTask() {
  log.info("Scheduled job is starting...");

  log.info("Running query to determine total user records...");
  String sql = "SELECT COUNT(*) FROM users";
  int size = jdbcTemplate.queryForObject(sql, Integer.class);
  log.info("{} total user record(s) found", size);

  log.info("Running query to determine total inactive user records...");
  sql = "SELECT COUNT(*) FROM users u WHERE u.enabled = ?";		
  size = jdbcTemplate.queryForObject(sql, new Object[]{ false }, Integer.class);
  log.info("{} total inactive user record(s) found", size);

  log.info("Running query to delete inactive user records...");
  sql = "DELETE FROM users u WHERE u.enabled = ?";
  jdbcTemplate.update(sql, new Object[]{ false });
  sql = "SELECT COUNT(*) FROM users";
  size = jdbcTemplate.queryForObject(sql, Integer.class);
  log.info("{} total user record(s) found after removing inactive users", size);
}
{% endhighlight %}

The `scheduledTask()` method will run 30 seconds past each minute. This method is annotated with `@Scheduled` which
tells the Spring Framework to run it repeatedly.  
The `cron` property takes a [`cron`][cron]{:target="_blank"} expression. In our situation it is an expression to run
30 seconds  past each minute.

The `@Scheduled` annotation can be added to a method along with trigger metadata. For example, the following method
would be invoked every 5 seconds with a fixed delay, meaning that the period will be measured from the completion time
of each preceding invocation.

{% highlight java %}
@Scheduled(fixedDelay=5000)
public void scheduledTask() {
  // something that should execute periodically
}
{% endhighlight %}

If a fixed rate execution is desired, simply change the property name specified within the annotation. The following 
would be executed every 5 seconds measured between the successive start times of each invocation.

{% highlight java %}
@Scheduled(fixedRate=5000)
public void scheduledTask() {
  // something that should execute periodically
}
{% endhighlight %}

For fixed-delay and fixed-rate tasks, an initial delay may be specified indicating the number of milliseconds to wait 
before the first execution of the method.

{% highlight java %}
@Scheduled(initialDelay=1000, fixedRate=5000)
public void scheduledTask() {
  // something that should execute periodically
}
{% endhighlight %}

> Notice that the methods to be scheduled must have void returns and must not expect any arguments. If the method needs
  to interact with other objects from the Application Context, then those would typically have been provided through
  dependency injection

To enable support for `@Scheduled` annotation add `@EnableScheduling` to one of your `@Configuration` classes:

{% highlight java %}
@EnableScheduling
@SpringBootApplication
public class Application {
  // insert logic here
}
{% endhighlight %}

Using the `@EnableScheduling` Spring will by default be searching for an associated scheduler definition: either a 
unique `org.springframework.scheduling.TaskScheduler` bean in the context, or a `TaskScheduler` bean named 
`"taskScheduler"` otherwise; the same lookup will also be performed for a 
`java.util.concurrent.ScheduledExecutorService` bean. If neither of the two is resolvable, a local single-threaded
default scheduler will be created and used within the registrar.  
We will create a `Bean` of type `TaskScheduler` named `taskScheduler`. The name of this bean will serve as the thread name prefix (`taskScheduler-`) of the thread created by the framework to handle the scheduled jobs.

{% highlight java %}
@Bean
public TaskScheduler taskScheduler(){
  return new ThreadPoolTaskScheduler();
}
{% endhighlight %}

Putting it all together for the scheduling we have:

file: `src/main/java/com/juliuskrah/Application.java`

{% highlight java %}
@EnableScheduling
@SpringBootApplication
public class Application {

  public static void main(String... args) {
    SpringApplication.run(Application.class, args);
  }

  @Bean
  public TaskScheduler taskScheduler(){
    return new ThreadPoolTaskScheduler();
  }
}
{% endhighlight %}

file: `src/main/java/com/juliuskrah/schedule/Schedule.java`

{% highlight java %}
@Slf4j
@Component
public class Schedule {

  @Autowired
  private JdbcTemplate jdbcTemplate;

  @Scheduled(cron = "30 0/1 * * * ?")
  public void scheduledTask() {
    log.info("Scheduled job is starting...");

    log.info("Running query to determine total user records...");
    String sql = "SELECT COUNT(*) FROM users";
    int size = jdbcTemplate.queryForObject(sql, Integer.class);
    log.info("{} total user record(s) found", size);

    log.info("Running query to determine total inactive user records...");
    sql = "SELECT COUNT(*) FROM users u WHERE u.enabled = ?";		
    size = jdbcTemplate.queryForObject(sql, new Object[]{ false }, Integer.class);
    log.info("{} total inactive user record(s) found", size);

    log.info("Running query to delete inactive user records...");
    sql = "DELETE FROM users u WHERE u.enabled = ?";
    jdbcTemplate.update(sql, new Object[]{ false });
    sql = "SELECT COUNT(*) FROM users";
    size = jdbcTemplate.queryForObject(sql, Integer.class);
    log.info("{} total user record(s) found after removing inactive users", size);
  }
}
{% endhighlight %}


# Task Executor
Sometimes you would want to perform a long running operation and you would not want to wait for the operation to 
`return` control before proceeding. You would rather prefer to carry on with other operations and be notified once
this long-running operation has `returned` at which point you would do other things with it.  
In a scenario like this you would want to perform this operation `asynchronously` and this is where Spring's `Task
Execution` comes into picture. 

In this example we will make a batch insert of one million records and record the execution time:

{% highlight java %}
@Async
public void insertBatch() {
  log.info("Starting bulk insert into users table asynchronously...");
  String sql = "INSERT INTO users VALUES(?, ?, ?)";
  StopWatch watch = new StopWatch();
  watch.start();

  jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter(){
    @Override
    public void setValues(PreparedStatement ps, int i) throws SQLException {
      ps.setString(1, String.format("user_%s", i));
      ps.setString(2, String.format("user123_%s", i));
      ps.setBoolean(3, i%3 == 0 ? false : true);
    }

    @Override
    public int getBatchSize() {
      return 1000000;
    }
  });
  watch.stop();
  log.info("1000000 records inserted asynchronously in {} s", watch.getTotalTimeSeconds());
}
{% endhighlight %}

The `@Async` annotation can be provided on a method so that invocation of that method will occur asynchronously. In 
other words, the caller will return immediately upon invocation and the actual execution of the method will occur in a 
task that has been submitted to a Spring `TaskExecutor`. In the simplest case, the annotation may be applied to a 
void-returning method.

{% highlight java %}
@Async
void insertBatch() {
  // this will be executed asynchronously
}
{% endhighlight %}

Unlike the methods annotated with the `@Scheduled` annotation, these methods can expect arguments, because they will be 
invoked in the "normal" way by callers at runtime rather than from a scheduled task being managed by the container. For 
example, the following is a legitimate application of the `@Async` annotation.

{% highlight java %}
@Async
void insertBatch(String s) {
  // this will be executed asynchronously
}
{% endhighlight %}

Even methods that return a value can be invoked asynchronously. However, such methods are required to have a `Future`
typed return value. This still provides the benefit of asynchronous execution so that the caller can perform other 
tasks prior to calling `get()` on that `Future`.

{% highlight java %}
@Async
Future<String> insertBatch(int i) {
  // this will be executed asynchronously
}
{% endhighlight %}

To enable support for `@Async` annotation add `@EnableAsync` to one of your `@Configuration` classes:

{% highlight java %}
@EnableAsync
@SpringBootApplication
public class Application {
  // insert logic here
}
{% endhighlight %}

Using the `@EnableAsnc` Spring will by default be searching for an associated thread pool definition: either a unique 
`org.springframework.core.task.TaskExecutor` bean in the context, or an `java.util.concurrent.Executor` bean named
`"taskExecutor"` otherwise. If neither of the two is resolvable, a
`org.springframework.core.task.SimpleAsyncTaskExecutor` will be used to process async method invocations.  
We will create a Bean of type `TaskExecutor` named `taskExecutor`. The name of this bean will serve as the thread name
prefix (`taskExecutor-`) of the thread created by the framework to handle the async jobs:

{% highlight java %}
@Bean
public TaskExecutor taskExecutor() {
  return new ThreadPoolTaskExecutor();
}
{% endhighlight %}

Putting it all together we have:

file: `src/main/java/com/juliuskrah/Application.java`

{% highlight java %}
@EnableAsync
@EnableScheduling
@SpringBootApplication
public class Application {
  @Autowired
  private Execute execute;

  public static void main(String... args) {
    SpringApplication.run(Application.class, args);
  }

  @Bean
  public TaskScheduler taskScheduler() {
    return new ThreadPoolTaskScheduler();
  }

  @Bean
  public TaskExecutor taskExecutor() {
    return new ThreadPoolTaskExecutor();
  }

  @PostConstruct
  public void init() {
    execute.insertBatch();
  }
}
{% endhighlight %}

file: `src/main/java/com/juliuskrah/execute/Execute.java`

{% highlight java %}
@Slf4j
@Component
public class Execute {
  @Autowired
  private JdbcTemplate jdbcTemplate;
   
  @Async
  public void insertBatch() {
    log.info("Starting bulk insert into users table asynchronously...");
    String sql = "INSERT INTO users VALUES(?, ?, ?)";
    StopWatch watch = new StopWatch();
    watch.start();

    jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
      @Override
      public void setValues(PreparedStatement ps, int i) throws SQLException {
        ps.setString(1, String.format("user_%s", i));
        ps.setString(2, String.format("user123_%s", i));
        ps.setBoolean(3, i%3 == 0 ? false : true);
      }

      @Override
      public int getBatchSize() {
        return 1000000;
      }
    });
    watch.stop();
    log.info("1000000 records inserted asynchronously in {} s", watch.getTotalTimeSeconds());
  }
}
{% endhighlight %}

When you run this code you should get an output similar to the following:

{% highlight bash %}
...
2017-06-03 00:04:26.206  INFO 13720 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService  'taskExecutor'
2017-06-03 00:04:26.253  INFO 13720 --- [ taskExecutor-1] com.juliuskrah.execute.Execute           : Starting bulk insert into users table asynchronously...
2017-06-03 00:04:26.255  INFO 13720 --- [           main] o.s.s.c.ThreadPoolTaskScheduler          : Initializing ExecutorService  'taskScheduler'
2017-06-03 00:04:26.940  INFO 13720 --- [           main] o.s.j.e.a.AnnotationMBeanExporter        : Registering beans for JMX exposure on startup
2017-06-03 00:04:26.975  INFO 13720 --- [           main] com.juliuskrah.Application               : Started Application in 3.28 seconds (JVM running for 14.123)
2017-06-03 00:04:30.001  INFO 13720 --- [taskScheduler-1] com.juliuskrah.schedule.Schedule         : Scheduled job is starting...
2017-06-03 00:04:30.001  INFO 13720 --- [taskScheduler-1] com.juliuskrah.schedule.Schedule         : Running query to determine total user records...
2017-06-03 00:04:30.021  INFO 13720 --- [taskScheduler-1] com.juliuskrah.schedule.Schedule         : 15 total user record(s) found
2017-06-03 00:04:30.024  INFO 13720 --- [taskScheduler-1] com.juliuskrah.schedule.Schedule         : Running query to determine total inactive user records...
2017-06-03 00:04:30.037  INFO 13720 --- [taskScheduler-1] com.juliuskrah.schedule.Schedule         : 4 total inactive user record(s) found
2017-06-03 00:04:30.038  INFO 13720 --- [taskScheduler-1] com.juliuskrah.schedule.Schedule         : Running query to delete inactive user records...
2017-06-03 00:04:30.060  INFO 13720 --- [taskScheduler-1] com.juliuskrah.schedule.Schedule         : 11 total user record(s) found after removing inactive users
2017-06-03 00:04:48.617  INFO 13720 --- [ taskExecutor-1] com.juliuskrah.execute.Execute           : 1000000 records inserted asynchronously in 22.359 s
2017-06-03 00:05:30.002  INFO 13720 --- [taskScheduler-1] com.juliuskrah.schedule.Schedule         : Scheduled job is starting...
2017-06-03 00:05:30.002  INFO 13720 --- [taskScheduler-1] com.juliuskrah.schedule.Schedule         : Running query to determine total user records...
2017-06-03 00:05:30.005  INFO 13720 --- [taskScheduler-1] com.juliuskrah.schedule.Schedule         : 1000011 total user record(s) found
2017-06-03 00:05:30.009  INFO 13720 --- [taskScheduler-1] com.juliuskrah.schedule.Schedule         : Running query to determine total inactive user records...
2017-06-03 00:05:30.584  INFO 13720 --- [taskScheduler-1] com.juliuskrah.schedule.Schedule         : 333334 total inactive user record(s) found
2017-06-03 00:05:30.593  INFO 13720 --- [taskScheduler-1] com.juliuskrah.schedule.Schedule         : Running query to delete inactive user records...
2017-06-03 00:05:35.404  INFO 13720 --- [       Thread-3] o.s.s.c.ThreadPoolTaskScheduler          : Shutting down ExecutorService 'taskScheduler'
2017-06-03 00:05:35.406  INFO 13720 --- [       Thread-3] o.s.s.concurrent.ThreadPoolTaskExecutor  : Shutting down ExecutorService 'taskExecutor'
{% endhighlight %}

# Conclusion
In this post we learned how to Schedule and Execute jobs leveraging the abstraction of the Spring Framework.  
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.

[cron]:                     https://en.wikipedia.org/wiki/Cron
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org