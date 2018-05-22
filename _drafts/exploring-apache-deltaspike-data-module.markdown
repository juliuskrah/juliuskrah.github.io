---
layout:     post
title:      Exploring Apache DeltaSpike Data Module
categories: blog
tags:       java maven javaee cdi jpa
section:    blog
author:     juliuskrah
repo:       cdi/tree/delta-spike
---
> Apache DeltaSpike

# Introduction

DeltaSpike Data Module... [Spring Data](/spring-data)

# Prerequisites

- [Java Development Kit][JDK]{:target="_blank"}
- [Maven][]{:target="_blank"}

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
|  |  |  |  |__templates/
|  |  |  |  |  |__default.xhtml
|  |  |  |  |__beans.xml
|  |  |  |  |__web.xml
|  |  |  |__index.html
|  |  |  |__index.xhtml
|__pom.xml
```

## Setting Up

Download the initial project ([zip](https://github.com/juliuskrah/cdi/archive/v2.0.zip)|
[tar.gz](https://github.com/juliuskrah/cdi/archive/v2.0.tar.gz)) and extract.

[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org