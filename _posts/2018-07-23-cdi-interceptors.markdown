---
layout:     post
title:      CDI Interceptors
date:       2018-07-23 23:24:55 +0000
categories: gist
tags:       java cdi javaee
section:    blog
author:     juliuskrah
repo:       cdi/tree/delta-spike
---
[CDI](/tag/cdi) Interceptor functionality is defined in the Java Interceptors specification.

The Interceptors specification defines three kinds of interception points:

- business method interception,
- lifecycle callback interception, and
- timeout method interception (EJB only).

We will explore the basics of using interceptors in CDI. A simple interceptor will be created that 
initializes a cutomer's account with 10 USD.

{% gist 52709b25f5e9fa9a71d135ace1ad9875 AccountOpeningInterceptor.java %}

In the above snippet we create the `@InterceptorBinding` which can be applied to a `TYPE` or a `METHOD`.

{% gist 52709b25f5e9fa9a71d135ace1ad9875 AccountCreditOpeningInterceptor.java %}

Next is to create the actual Interceptor class with a binding to the `AccountOpeningInterceptor`.

{% gist 52709b25f5e9fa9a71d135ace1ad9875 CustomerService.java %}

Then we apply the Interceptor to the method call that requires interception.

On that note and until the next post; keep doing cool things :smile:.