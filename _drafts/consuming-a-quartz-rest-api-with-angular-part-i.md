---
layout:     series
title:      Consuming a Quartz REST API with Angular Part 1
categories: tutorial
tags:       typescript npm angular quartz
section:    series
author:     juliuskrah
repo:       quartz-manager-ui/tree/master
---
> [Angular][]{:target="_blank"} is a platform that makes it easy to build applications with the web. 
  Angular combines declarative templates, dependency injection, end to end tooling, and integrated best practices to 
  solve development challenges. Angular empowers developers to build applications that live on the web, mobile, or the 
  desktop

# Introduction
In this post we will create a user interfaces to interract with the REST API we created in the previous 
[post]({% post_url 2017-10-11-error-handling-in-a-rest-service-with-quartz %}) of this 
[series]({% post_url 2017-09-26-dynamic-job-scheduling-with-quartz-and-spring %}).

## Project Structure
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
|  |  |  |  |  |  |__AutowiringSpringBeanJobFactory.java 
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
|  |  |__resources/
|  |  |  |  |__application.yaml
|__pom.xml
```

# Prerequisites
To follow along this guide, you should have the following:

- [Node][Node]{:target="_blank"} => 6.9.0 / [NPM][]{:target="_blank"} => 3 
- [TypeScript][]{:target="_blank"}

# Install and Setup Angular project
To ease developing an Angular project, we will use the [Angular CLI][CLI]{:target="_blank"}. First install the CLI:

```posh
C:\> npm install -g @angular/cli
```

Test the installation succeeded:

```posh
C:\> ng help
ng build <options...>
  Builds your app and places it into the output path (dist/ by default).
  aliases: b
  --target (String) (Default: development) Defines the build target.
    aliases: -t <value>, -dev (--target=development), -prod (--target=production), --target <value>
  --environment (String) Defines the build environment.
    aliases: -e <value>, --environment <value>
  --output-path (Path) Path where output will be placed.
    aliases: -op <value>, --outputPath <value>
  --aot (Boolean) Build using Ahead of Time compilation.
    aliases: -aot
...
ng xi18n <options...>
  Extracts i18n messages from source code.
  --i18n-format (String) (Default: xlf) Output format for the generated file.
    aliases: -f <value>, -xmb (--i18n-format=xmb), -xlf (--i18n-format=xlf), --xliff (--i18n-format=xlf), --i18nFormat <value>
  --output-path (Path) (Default: null) Path where output will be placed.
    aliases: -op <value>, --outputPath <value>
  --verbose (Boolean) (Default: false) Adds more details to output logging.
    aliases: --verbose
  --progress (Boolean) (Default: true) Log progress to the console while running.
    aliases: --progress
  --app (String) Specifies app name to use.
    aliases: -a <value>, -app <value>
  --locale (String) Specifies the source language of the application.
    aliases: -l <value>, --locale <value>
  --out-file (String) Name of the file to output.
    aliases: -of <value>, --outFile <value>
```

Now we create the Angular project:

```posh
C:\> ng new quartz-manager-ui
C:\> cd quartz-manager-ui
C:\> ng serve -o
```

Navigate to <http://localhost:4200/>{:target="_blank"}. The app will automatically reload if you change any of the source files.

![Angular Initial](https://i.imgur.com/Ki7A1gB.png)

{:.image-caption}
*Initial Angular Project*

## Create Services

[Angular]:                          https://angular.io/
[CLI]:                              https://cli.angular.io/
[Node]:                             https://nodejs.org/en/download/
[NPM]:                              https://www.npmjs.com/get-npm
[TypeScript]:                       https://www.typescriptlang.org/