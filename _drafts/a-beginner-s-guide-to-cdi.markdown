---
layout:     post
title:      A beginner's guide to CDI
categories: blog
tags:       java maven javaee cdi rest jax-rs jersey jpa
section:    blog
author:     juliuskrah
repo:       cdi/tree/2018.5.0
---
> [Contexts and Dependency Injection for the Java EE Platform][CDI]{:target="_blank"} is a 
  [JCP][]{:target="_blank"} standard for dependency injection and contextual lifecycle management and one of 
  the most important and popular parts of [Java EE](/tag/javaee/).

# Introduction

Contexts and Dependency Injection for Java EE (CDI) was introduced as part of the Java EE platform, and 
has quickly become one of the most important and popular components of the platform.

The CDI specification defines a set of complementary services that help improve the structure of application 
code. CDI layers an enhanced lifecycle and interaction model over existing Java component types, including 
managed beans and `Enterprise Java Beans`. 

You can run CDI in a Java EE or SE environment. However to leverage the full capabilities of CDI, it is 
recommended to use a Java EE container. In this tutorial, we will cover just the EE approach and we will be
using `Wildfly-Swarm`, a lighweight alternative to the Wildfly Java EE container which is also a very good
fit for deploying microservices.

## What We will Cover

CDI is quite broad; as it provides integration and contracts with the wider Java EE ecosystem. This being an 
introductory blog, we limit the scope of what is covered to:

- Bean Scope
- Producers (Field and Method)
- Injection Types (Field, Method, Constructor)
- Events

## Prerequisites

- [Java Development Kit][JDK]{:target="_blank"}
- [Maven][]{:target="_blank"}
- [Eclipse][]{:target="_blank"} 4.6.0+

## Project Structure

At the end of this guide our folder structure will look similar to the following:

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__juliuskrah/
|  |  |  |  |  |__cdi/
|  |  |  |  |  |  |__ApplicationContext.java
|  |  |  |  |  |  |__ApplicationResources.java
|  |  |  |  |  |  |__business/
|  |  |  |  |  |  |  |__CustomerService.java
|  |  |  |  |  |  |  |__dto/
|  |  |  |  |  |  |  |  |__CustomerBean.java
|  |  |  |  |  |  |  |__mapper/
|  |  |  |  |  |  |  |  |__CustomerMapper.java
|  |  |  |  |  |  |__entity/
|  |  |  |  |  |  |  |__Customer.java
|  |  |  |  |  |  |__repository/
|  |  |  |  |  |  |  |__CustomerRepository.java
|  |  |  |  |  |  |__web/
|  |  |  |  |  |  |  |__IndexController.java
|  |  |__resources/
|  |  |  |__db/
|  |  |  |  |__migration/
|  |  |  |  |  |__`V1__Create_customer_table.sql`
|  |  |  |__META-INF/
|  |  |  |  |__persistence.xml
|  |  |  |__modules/
|  |  |  |  |__com/
|  |  |  |  |  |__h2database/
|  |  |  |  |  |  |__h2/
|  |  |  |  |  |  |  |__main/
|  |  |  |  |  |  |  |  |__module.xml
|  |  |  |__project-defaults.yaml
|  |  |__webapp/
|  |  |  |__WEB-INF/
|  |  |  |  |__views/
|  |  |  |  |  |__index.xhtml
|  |  |  |  |  |__templates/
|  |  |  |  |  |  |__default.xhtml
|__pom.xml
```

## Setting Up

Download the initial project and extract. Import into Eclipse and follow the Mapstruct instructions here to setup in Eclipse.

[CDI]:                      http://cdi-spec.org/
[Eclipse]:                  https://www.eclipse.org/downloads/
[JCP]:                      https://jcp.org/en/home/index
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org
