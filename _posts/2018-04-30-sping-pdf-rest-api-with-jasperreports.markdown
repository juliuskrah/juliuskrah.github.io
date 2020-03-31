---
layout:     post
title:      Sping PDF REST API with JasperReports
date:       2018-04-30 18:33:45 +0000
categories: blog
tags:       java maven jasperreports rest spring-boot
section:    blog
author:     juliuskrah
repo:       jasper-html-mail/tree/jasper-rest
---
> [JasperReports](/tag/jasperreports/) is a Java class library, and it is meant for those Java developers
  who need to add reporting capabilities to their applications.

# Introduction

In this post, we will create a PDF report that is exposed over a RESTful endpoint using Spring Boot. This is also
a good place to start for those of you trying to figure out how to integrate JasperReports with Spring Framework 5. To serve excel over REST, [refer to this Excel Post]({% post_url 2020-03-27-sping-excel-rest-api-with-jasperreports %}).

## Prerequisites

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
|  |  |  |  |  |__jasper/
|  |  |  |  |  |  |__Application.java
|  |  |  |  |  |  |__ApplicationProperties.java
|  |  |  |  |  |  |__report/
|  |  |  |  |  |  |  |__JasperReportsService.java
|  |  |  |  |  |  |  |__ReportService.java
|  |  |  |  |  |  |__storage/
|  |  |  |  |  |  |  |__FileSystemStorageService.java
|  |  |  |  |  |  |  |__StorageException.java
|  |  |  |  |  |  |  |__StorageFileNotFoundException.java
|  |  |  |  |  |  |  |__StorageService.java
|  |  |  |  |  |  |__web/
|  |  |  |  |  |  |  |__rest/
|  |  |  |  |  |  |  |  |__ApplicationResource.java
|  |  |__resources/
|  |  |  |__reports/
|  |  |  |  |__pdf_rest_resource.jrxml
|  |  |  |  |__pdf_rest.jrxml
|  |  |  |__application.yaml
|  |  |  |__cherry.png
|  |  |  |__logo.png
|__pom.xml
```

## How to complete this guide

To complete this guide, download ([zip](https://github.com/juliuskrah/jasper-html-mail/archive/v2.0.zip)|
[tar.gz](https://github.com/juliuskrah/jasper-html-mail/archive/v2.0.tar.gz)) 
and unzip the source repository for this guide.

## Create PDF Report

Let us start by implementing a solution to generate PDFs in JasperReports:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/jasper/report/JasperReportsService.java' %}

{% highlight java %}
@Component
public class JasperReportsService implements ReportService {
  // ...
  @Override
  public byte[] generatePDFReport(String inputFileName, Map<String, Object> params,
    JRDataSource dataSource) {
    byte[] bytes = null;
    JasperReport jasperReport = null;
    try (ByteArrayOutputStream byteArray = new ByteArrayOutputStream()) {
      // Check if a compiled report exists
      if (storageService.jasperFileExists(inputFileName)) {
        jasperReport = (JasperReport) JRLoader.loadObject(storageService.loadJasperFile(inputFileName));
      }
      // Compile report from source and save
      else {
        String jrxml = storageService.loadJrxmlFile(inputFileName);
        jasperReport = JasperCompileManager.compileReport(jrxml);
        // Save compiled report. Compiled report is loaded next time
        JRSaver.saveObject(jasperReport, storageService.loadJasperFile(inputFileName));
      }
      JasperPrint jasperPrint = JasperFillManager.fillReport(jasperReport, params, dataSource);
      // return the PDF in bytes
      bytes = JasperExportManager.exportReportToPdf(jasperPrint);
    }
    catch (JRException | IOException e) {
      e.printStackTrace();
    }
    return bytes;
  }
}
{% endhighlight %}

## Render the PDF REST Endpoint

We will create the REST endpoint:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/jasper/web/rest/ApplicationResource.java' %}

{% highlight java %}
@RestController
public class ApplicationResource {
  // ...
  @GetMapping("/{username}")
  public ResponseEntity<byte[]> report(@PathVariable(required = false) String username) {
    Map<String, Object> params = new HashMap<>();
    params.put("username", username);
    byte[] bytes = reportService.generatePDFReport("pdf_rest_resource", params);
    return ResponseEntity
      .ok()
      // Specify content type as PDF
      .header("Content-Type", "application/pdf; charset=UTF-8")
      // Tell browser to display PDF if it can
      .header("Content-Disposition", "inline; filename=\"" + username + ".pdf\"")
      .body(bytes);
  }
}
{% endhighlight %}

Now navigate to <http://localhost:8080/:username> to view your report.

# Conclusion

In this post we learned how to generate PDF with JasperReports and serve it over HTTP with Spring Boot.      
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool 
things :+1:.

[JasperReports]:            https://community.jaspersoft.com/project/jasperreports-library
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org