---
layout:     series
title:      Dynamic Job Scheduling with Quartz and Spring
categories: tutorial
tags:       java maven liquibase quartz rest spring-boot
section:    series
author:     juliuskrah
repo:       quartz-manager/tree/master
---
> [Quartz Scheduler][Quartz]{:target="_blank"} is a richly featured, open source job scheduling library that can
  be integrated within virtually any Java application - from the smallest stand-alone application to the largest
  e-commerce system.

# Introduction
Every developer at a certain point in his carreer is faced with the difficult task of scheduling jobs dynamically. 
In this post we are going to create a sample application for dynamically scheduling jobs using a
[REST API]({% post_url 2017-07-16-developing-restful-services-with-jax-rs-jersey %}).  
We will dynamically create jobs that sends emails to a predefined group of people on a user defined schedule.

## Concepts
Before we dive any further, there are a few quartz concepts we need to understand:

1.    Job - an interface to be implemented by components that you wish to have executed by the scheduler. The 
      interface has one method `execute(...)`. This is where your scheduled task runs. Information on the 
      `JobDetail` and `Trigger` is retrieved using the `JobExecutionContext`.

      ```java
      package org.quartz;

      public interface Job {
        public void execute(JobExecutionContext context) throws JobExecutionException;
      }
       ```

2.    JobDetail - used to define instances of `Jobs`. This defines how a job is run. Whatever 
      data you want available to the `Job` when it is instantiated is provided through the `JobDetail`.  
      Quartz provides a Domain Specific Language (DSL) in the form of `JobBuilder` for constructing `JobDetail`
      instances.

      ```java
      // define the job and tie it to the Job implementation
      JobDetail job = newJob(HelloJob.class)
        .withIdentity("myJob", "group1") // name "myJob", group "group1"
        .build();
      ```

3.    Trigger - a component that defines the schedule upon which a given Job will be executed. The trigger  
      provides instruction on when the job is run.  
      Quartz provides a DSL (TriggerBuilder) for constructing `Trigger` instances.

      ```java
      // Trigger the job to run now, and then every 40 seconds
      Trigger trigger = newTrigger()
        .withIdentity("myTrigger", "group1")
        .startNow()
        .withSchedule(simpleSchedule()
          .withIntervalInSeconds(40)
          .repeatForever())            
        .build();
      ```

4.    Scheduler - the main API for interacting with the scheduler. A **Scheduler’s** life-cycle is bounded by it’s
      creation, via a **SchedulerFactory** and a call to its `shutdown()` method. Once created the Scheduler 
      interface can be used _add_, _remove_, and _list_ `Jobs` and `Triggers`, and perform other
      scheduling-related operations (such as _pausing_ a trigger). However, the Scheduler will not actually act on
      any triggers (execute jobs) until it has been started with the `start()` method.

      ```java
      SchedulerFactory schedFact = new org.quartz.impl.StdSchedulerFactory();

      Scheduler sched = schedFact.getScheduler();
      sched.start();

      // Tell quartz to schedule the job using our trigger
      sched.scheduleJob(job, trigger);
      ```

# Create and Setup Dependencies for the Sample Application


[Quartz]:               http://www.quartz-scheduler.org/