---
layout:     post
title:      Sping Excel REST API with JasperReports
date:       2020-03-27 13:52:08 +0000
categories: blog
tags:       java maven jasperreports rest spring-boot
section:    blog
author:     juliuskrah
repo:       jasper-html-mail/tree/jasper-rest-multi
---
> [JasperReports](/tag/jasperreports/) is a Java library, and it is meant for Java developers
  who need to add reporting to their applications.

# Introduction
In this post, we will create an excel report that is exposed over RESTful endpoint using Spring Boot. In
a [previous post]({% post_url 2018-04-30-sping-pdf-rest-api-with-jasperreports %}) we looked at how to
expose Jasper PDF over REST.

## Prerequisites

- [Java Development Kit][JDK]{:target="_blank"} 11+
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

To complete this guide, download ([zip](https://github.com/juliuskrah/jasper-html-mail/archive/v3.0.zip)|
[tar.gz](https://github.com/juliuskrah/jasper-html-mail/archive/v3.0.tar.gz)) 
and unzip the source repository for this guide.

## Create Excel Report

Let us start by implementing a solution to generate excel in JasperReports:

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
    try {
      if (storageService.jasperFileExists(inputFileName)) {
        jasperReport = (JasperReport) JRLoader.loadObject(storageService.loadJasperFile(inputFileName));
      } else {
        var jrxml = storageService.loadJrxmlFile(inputFileName);
        jasperReport = JasperCompileManager.compileReport(jrxml);
        // Save the compiled report for use on next invocation
        JRSaver.saveObject(jasperReport, storageService.loadJasperFile(inputFileName));
      }
      JasperPrint jasperPrint = JasperFillManager.fillReport(jasperReport, params, dataSource);
      var input = new SimpleExporterInput(jasperPrint);
      try (var byteArray = new ByteArrayOutputStream()) {
        var output = new SimpleOutputStreamExporterOutput(byteArray);
        var exporter = new JRXlsxExporter();
        exporter.setExporterInput(input);
        exporter.setExporterOutput(output);
        exporter.exportReport();
        bytes = byteArray.toByteArray();
        output.close();
      } catch (IOException e) {
        // handle IO error
      }
      return bytes;
    } catch (JRException e) {
      // handle error
    }
    return bytes;
  }
}
{% endhighlight %}

## Render the Excel REST Endpoint

We will create the REST endpoint:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/jasper/web/rest/ApplicationResource.java' %}

{% highlight java %}
@RestController
public class ApplicationResource {
  // ...
  @GetMapping("/{username}")
  public ResponseEntity<byte[]> report(@PathVariable String username) {
    Map<String, Object> params = new HashMap<>();
    params.put("username", username);
    byte[] bytes = reportService.generatePDFReport("pdf_rest_resource", params);
    var contentDisposition = ContentDisposition.builder("attachment")
      .filename(username + ".xlsx").build();
    HttpHeaders headers = new HttpHeaders();
    headers.setContentDisposition(contentDisposition);
    return ResponseEntity
      .ok()
      // Specify content type as excel
      .header("Content-Type", "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet; charset=UTF-8")
      // Tell browser to download Excel if it can
      .headers(headers) //
      .body(bytes);
  }
}
{% endhighlight %}

Now navigate to <http://localhost:8080/:username> to view your report.

# Conclusion

In this post we learned how to generate Excel with JasperReports and serve it over HTTP with Spring
Boot. As usual you can find the full example to this guide {% include source.html %}. Until the next
post, keep doing cool things :+1:.

[JasperReports]:            https://community.jaspersoft.com/project/jasperreports-library
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org
