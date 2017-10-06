---
layout:     series
title:      Persisting Dynamic Jobs with Quartz and Spring
date:       2017-10-06 23:11:17 +0000
categories: tutorial
tags:       java maven liquibase quartz rest spring-boot
section:    series
author:     juliuskrah
repo:       quartz-manager/tree/v1.x
---
> [Quartz Scheduler][]{:target="_blank"} is a richly featured, open source job scheduling library that can be 
  integrated within virtually any Java application - from the smallest stand-alone application to the largest 
  e-commerce system.

# Introduction
Welcome to the second post in the _Dynamic Scheduling with Quartz_ series. In the previous post I talked about 
scheduling jobs dynamically and storing then in memory with [RAMJobStore]({% post_url 2017-09-26-dynamic-job-scheduling-with-quartz-and-spring %}).
Although this approach is performant, the jobs do not survive application crash or a restart.

In this post, I will show you how to create peristent jobs that are stored in a Relational Database. If I get
enough requests, I will do another post that stores the jobs in a Non-Relational Database.

To store jobs in a relational database with quartz the `JDBCJobStore` is used. In the next section I will
delve deeper into the JDBCJobStore.

## The JDBCJobStore
JobStore's are responsible for keeping track of all the "work data" that you give to the scheduler: `jobs`, `triggers`,
`calendars`, etc. Selecting the appropriate JobStore for your Quartz scheduler instance is an important step. Luckily,
the choice should be a very easy one once you understand the differences between them.

In this post I will talk about the `JDBCJobStore` and persistence in a relational database using 
[`Liquibase`]({% post_url 2017-02-26-database-migration-with-liquibase-hikaricp-hibernate-and-jpa %}) for database
migration.

`JDBCJobStore` keeps all of its data in a database via JDBC. Because of this it is a bit more complicated to configure
than `RAMJobStore`, and it also is not as fast. However, the performance draw-back is not terribly bad, especially if
you build the database tables with indexes on the primary keys. On fairly modern set of machines with a decent LAN 
(between the scheduler and database) the time to retrieve and update a firing trigger will typically be less than 10 milliseconds.

JDBCJobStore works with nearly any database, it has been used widely with `Oracle`, `PostgreSQL`, `MySQL`, 
`MS SQLServer`, `HSQLDB`, and `DB2`. In this post we will be using `HSQLDB` as the persistence backend. This
can be adapted to fit any relational database of your choosing.  
To use JDBCJobStore, you must first create a set of database tables for Quartz to use. You can find table-creation SQL 
scripts in the "[org/quartz/impl/jdbcjobstore](https://github.com/quartz-scheduler/quartz/tree/master/quartz-core/src/main/resources/org/quartz/impl/jdbcjobstore){:target="_blank"}" 
directory of the Quartz distribution. If there is not already a script for your database type, just look at one of the
existing ones, and modify it in any way necessary for your DB. One thing to note is that in these scripts, all the the
tables start with the prefix "`QRTZ_`" (such as the tables "`QRTZ_TRIGGERS`", and "`QRTZ_JOB_DETAIL`"). This prefix 
can actually be anything you'd like, as long as you inform JDBCJobStore what the prefix is 
(in your Quartz properties). Using different prefixes may be useful for creating multiple sets of tables, for multiple
scheduler instances, within the same database.

There are two seperate JDBCJobStore classes that you can select between, depending on the transactional behaviour you
need:-  

1.  **JobStoreTX** - manages all transactions itself by calling `commit()` (or `rollback()`) on the database 
    connection after every action (such as the addition of a job). `JobStoreTX` is appropriate if you are using Quartz 
    in a stand-alone application, or within a servlet container if the application is not using JTA transactions.
    The JobStoreTX is selected by setting the '`org.quartz.jobStore.class`' property in the `quartz.properties`:

    ```
    org.quartz.jobStore.class = org.quartz.impl.jdbcjobstore.JobStoreTX
    ```
2.  **JobStoreCMT** - relies upon transactions being managed by the application which is using Quartz. A JTA 
    transaction must be in progress before attempt to schedule (or unschedule) jobs/triggers. This allows the "work"
    of scheduling to participate in a global transaction. `JobStoreCMT` actually requires the use of two datasources
    - one that has it's connection's transactions managed by the application server (via JTA) and 
    - one datasource that has connections that do not participate in global (JTA) transactions. 
    
    JobStoreCMT is appropriate when applications are using JTA transactions (such as via EJB Session Beans) to perform
    their work.

    The JobStore is selected by setting the '`org.quartz.jobStore.class`' property  in the `quartz.properties`:

    ```
    org.quartz.jobStore.class = org.quartz.impl.jdbcjobstore.JobStoreCMT
    ```

Once you have decided on the type of `JDBCJobStore` to use based on your transactional needs, you need to select a
`DriverDelegate` for the JobStore to use. The DriverDelegate is responsible for doing any JDBC work that may be needed
for your specific database. `StdJDBCDelegate` is a delegate that uses "vanilla" JDBC code (and SQL statements) to do 
its work. If there isn't another delegate made specifically for your database, try using this delegate. Other 
delegates can be found in the "`org.quartz.impl.jdbcjobstore`" package, or in its sub-packages. Other delegates 
include `DB2v6Delegate` (for DB2 version 6 and earlier), `HSQLDBDelegate` (for HSQLDB), `MSSQLDelegate` (for Microsoft
SQLServer), `PostgreSQLDelegate` (for PostgreSQL), `WeblogicDelegate` (for using JDBC drivers made by Weblogic), 
`OracleDelegate` (for using Oracle), and others.

Once you've selected your delegate, set its class name as the delegate for JDBCJobStore to use:

```
org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.StdJDBCDelegate
```

# Directory structure
At the end of this guide our folder structure will look similar to the following:

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__juliuskrah/
|  |  |  |  |  |__quartz/
|  |  |  |  |  |  |__Application.java
|  |  |  |  |  |  |__job/
|  |  |  |  |  |  |  |__EmailJob.java
|  |  |  |  |  |  |__model/
|  |  |  |  |  |  |  |__JobDescriptor.java
|  |  |  |  |  |  |  |__TriggerDescriptor.java
|  |  |  |  |  |  |__service/
|  |  |  |  |  |  |  |__EmailService.java
|  |  |  |  |  |  |__web/
|  |  |  |  |  |  |  |__rest/
|  |  |  |  |  |  |  |  |__EmailResource.java
|  |  |  |__org/
|  |  |  |  |__springframework/
|  |  |  |  |  |__boot/ 
|  |  |  |  |  |  |__autoconfigure/  
|  |  |  |  |  |  |  |__quartz/
|  |  |  |  |  |  |  |  |__AutowireCapableBeanJobFactory.java  
|  |  |__resources/
|  |  |  |__db/
|  |  |  |  |__changelog/
|  |  |  |  |  |__db.changelog-master.yaml
|  |  |  |__application.yaml
|  |  |  |__quartz.properties
|__pom.xml
```

## Prerequisites
To follow alongside this guide by implementing the code, you should have the following set up:

- [Java Development Kit][JDK]{:target="_blank"}

_Optional_

- [Maven][]{:target="_blank"}
- [cURL][]{:target="_blank"}

# Getting the project
This post builds upon the [previous post]({% post_url 2017-09-26-dynamic-job-scheduling-with-quartz-and-spring %})
and you can get the source/base for this post as [zip](https://github.com/juliuskrah/quartz-manager/archive/v1.0.zip)|[tar.gz](https://github.com/juliuskrah/quartz-manager/archive/v1.0.tar.gz).

Extract the contents somewhere on your development system and let us begin.

## Preparing Liquibase Migration Scripts
We will setup our liquibase migration script:

file: {% include file-path.html file_path='src/main/resources/db/changelog/db.changelog-master.yaml' %}

{% highlight yaml %}
databaseChangeLog:
- changeSet:
    id: 1506785516870-1
    author: Julius
    ...
    changes:
    - createTable:
        columns:
        - column:
            constraints:
              nullable: false
            name: SCHED_NAME
            type: VARCHAR(120)
        - column:
            constraints:
              nullable: false
            name: TRIGGER_NAME
            type: VARCHAR(200)
        - column:
            constraints:
              nullable: false
            name: TRIGGER_GROUP
            type: VARCHAR(200)
        - column:
            name: BLOB_DATA
            type: BLOB
        tableName: QRTZ_BLOB_TRIGGERS
        ...
{% endhighlight %}

And tell Spring to run the migration on application initialization:

file: {% include file-path.html file_path='src/main/resources/application.yaml' %}

{% highlight yaml %}
liquibase:
  enabled: true
  ...
{% endhighlight %}

## Setup SchedulerFactoryBean
We will create a bean of type `SchedulerFactoryBean` where we will setup a JDBCJobStore:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/Application.java' %}

{% highlight java %}
...
@Bean
public SchedulerFactoryBean schedulerFactory(ApplicationContext applicationContext, DataSource dataSource) {
  SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
  schedulerFactoryBean.setDataSource(dataSource);
  schedulerFactoryBean.setConfigLocation(new ClassPathResource("quartz.properties"));
  schedulerFactoryBean.setJobFactory(
    new AutowireCapableBeanJobFactory(applicationContext.getAutowireCapableBeanFactory()));
  return schedulerFactoryBean;
}
{% endhighlight %}

In the snippet above we are setting the `Datasource`. When we do this, we are actually setting a JobStoreCMT of
type `org.springframework.scheduling.quartz.LocalDataSourceJobStore`.

- **LocalDataSourceJobStore** - Subclass of Quartz's JobStoreCMT class that delegates to a Spring-managed
  DataSource instead of using a Quartz-managed connection pool. This JobStore will be used if SchedulerFactoryBean's
  "dataSource" property is set.
 
  Supports both transactional and non-transactional DataSource access. With a non-XA DataSource and local Spring 
  transactions, a single DataSource argument is sufficient.

  Operations performed by this JobStore will properly participate in any kind of Spring-managed transaction, as it 
  uses Spring's DataSourceUtils connection handling methods that are aware of a current transaction.

  Note that all Quartz Scheduler operations that affect the persistent job store should usually be performed within 
  active transactions, as they assume to get proper locks etc.

We need a `quartz.properties` on the classpath to set the `org.quartz.impl.jdbcjobstore.DriverDelegate` for HSQLDB:

file: {% include file-path.html file_path='src/main/resources/quartz.properties' %}

{% highlight properties %}
org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.HSQLDBDelegate
{% endhighlight %}

We are all set. Now you can schedule dynamic jobs and these jobs will be saved in the database and persisted across
application restarts.

In the following sections I will show you how to optimize threads and jobs.

## Using Spring Task Execution Abstraction
Long-running jobs prevent others from running (if all threads in the ThreadPool are busy).

If you feel the need to call Thread.sleep() on the worker thread executing the Job, it is typically a sign that the
job is not ready to do the rest of its work because it needs to wait for some condition (such as the availability of a 
data record) to become true.

A better solution is to release the worker thread (exit the job) and allow other jobs to execute on that thread. _The 
job can reschedule itself, or other jobs before it exits_.

We will setup up a service class that runs `async` methods using
[Spring TaskExecutor]({% post_url 2017-06-03-task-execution-and-scheduling-with-spring %}) abstraction:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/service/AsyncMailSender.java' %}

{% highlight java %}
@Service
public class AsyncMailSender {
  private final JavaMailSender mailSender;

  public AsyncMailSender(JavaMailSender mailSender) {
    this.mailSender = mailSender;
  }
	
  @Async
  public void send(MimeMessage message) {
    mailSender.send(message);
  }
}
{% endhighlight %}

To understand the `async` annotation take a look at this [article]({% post_url 2017-06-03-task-execution-and-scheduling-with-spring %}).

For Spring to process all `Async` annotations, we need to update the configuration class to `EnableAsync`:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/Application.java' %}

{% highlight java %}
@EnableAsync
@SpringBootApplication
public class Application implements AsyncConfigurer {
  ...

  @Override
  @Bean(name = "taskExecutor")
  public Executor getAsyncExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(5);
    executor.setMaxPoolSize(6);
    executor.setQueueCapacity(100);
    executor.setThreadNamePrefix("email-");
    return executor;
  }

  @Override
  public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
    return new SimpleAsyncUncaughtExceptionHandler();
  }
}
{% endhighlight %}

Quartz by default uses a `org.quartz.simpl.SimpleThreadPool` for creating scheduling and executing jobs. We will
override this behaviour and ask Quartz to share in the thread-pool created by Spring:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/Application.java' %}

{% highlight java %}
...
@Bean
public SchedulerFactoryBean schedulerFactory(ApplicationContext applicationContext, 
            DataSource dataSource, Executor taskExecutor) {
  ...
  schedulerFactoryBean.setTaskExecutor(taskExecutor);
  return schedulerFactoryBean;
}
{% endhighlight %}

The above delegates all jobs to the `org.springframework.scheduling.quartz.LocalTaskExecutorThreadPool` when executing
jobs.

Use the `AsyncMailSender` in the `EmailJob`:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/quartz/job/EmailJob.java' %}

{% highlight java %}
public class EmailJob implements Job {
  @Autowired
  private AsyncMailSender asyncMailSender;
  ...
  private void sendEmail() {
    MimeMessage message = mailSender.createMimeMessage();
    ...
    asyncMailSender.send(message);
  }
}
{% endhighlight %}

That's all folks.

# Conclusion
In this post we learned how to schedule quartz jobs dynamically using `JDBCJobStore`. We also covered how do use
`Async` annotation to make our jobs execute faster.

You can find the source to this guide {% include source.html %}. Until the next post, keep doing cool things :smile:.

[cURL]:                                       https://curl.haxx.se/
[JDK]:                                        http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                                      http://maven.apache.org
[Quartz Scheduler]:                           http://www.quartz-scheduler.org/
