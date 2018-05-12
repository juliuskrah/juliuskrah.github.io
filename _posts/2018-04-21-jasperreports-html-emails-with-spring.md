---
layout:     post
title:      JasperReports HTML Emails with Spring
date:       2018-04-21 13:47:17 +0000
categories: blog
tags:       java maven jasperreports spring-boot
section:    blog
author:     juliuskrah
repo:       jasper-html-mail/tree/jasper-mail
---
> [JasperReports][]{:target="_blank"} is a library entirely written in Java and it is able to use data coming from
  any kind of data source and produce pixel-perfect documents that can be viewed, printed or exported in a variety of
  document formats including HTML, PDF, Excel, OpenOffice and Word.

# Introduction

We will look at two types of HTML emails that can be sent with Spring's abstraction of `JavaMail`:

1. HTML email with no inline
2. HTML email with inline (using image resources)

Generally when sending emails as HTML with Spring, you can use `Thymeleaf`, `Freemaker` etc to template your HTML. 
In this post we will look at how to achieve the same thing using `JasperReports`.

What we will _not_ be doing in this post is how to send attachments with Spring's abstraction of `JavaMail`.

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
|  |  |  |  |  |  |__mail/
|  |  |  |  |  |  |  |__EmailService.java
|  |  |  |  |  |  |  |__HtmlEmailService.java
|  |  |  |  |  |  |  |__JasperReportsService.java
|  |  |  |  |  |  |  |__ReportService.java
|  |  |  |  |  |  |__storage/
|  |  |  |  |  |  |  |__FileSystemStorageService.java
|  |  |  |  |  |  |  |__StorageException.java
|  |  |  |  |  |  |  |__StorageFileNotFoundException.java
|  |  |  |  |  |  |  |__StorageService.java
|  |  |__resources/
|  |  |  |__reports/
|  |  |  |  |__html_inline.jrxml
|  |  |  |  |__html.jrxml
|  |  |  |__application.yaml
|  |  |  |__cherry.png
|  |  |  |__logo.png
|__pom.xml
```

## How to complete this guide

To complete this guide, download ([zip](https://github.com/juliuskrah/jasper-html-mail/archive/v1.0.zip)|
[tar.gz](https://github.com/juliuskrah/jasper-html-mail/archive/v1.0.tar.gz)) 
and unzip the source repository for this guide. You also need to set the following environment variables:

1. MAIL_HOST
2. MAIL_PASSWORD
3. MAIL_PORT
4. MAIL_USERNAME

<script src="https://asciinema.org/a/177759.js" id="asciicast-177759" data-rows="24" data-t="20ss" data-autoplay="0" async></script>

# HTML Mail without Inline

Download and extract the base project if you haven't already done so. Create the class `HtmlEmailService` that
implements the `EmailService`. We will implement `sendHtmlEmail()` for the time being:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/jasper/mail/HtmlEmailService.java' %}

{% highlight java %}
@Async
@Component
public class HtmlEmailService implements EmailService {
  private final JavaMailSender javaMail;
  private final ApplicationProperties properties;

  public HtmlEmailService(JavaMailSender javaMail, ApplicationProperties properties) {
    this.javaMail = javaMail;
    this.properties = properties;
  }

  @Override
  public void sendHtmlEmail(String recipient, String html) {
    final MimeMessage message = javaMail.createMimeMessage();
    try {
      final MimeMessageHelper helper = new MimeMessageHelper(message, "UTF-8");
      helper.setFrom(properties.getMail().getSender(),
        properties.getMail().getPersonal());
      helper.setTo(recipient);
      helper.setSubject(properties.getMail().getMessageSubject());
      // Set to true for HTML
      helper.setText(html, true);
      javaMail.send(message);
    } catch (MessagingException | UnsupportedEncodingException e) {
      e.printStackTrace();
    }
  }

  // Implementation details omitted for brevity
}
{% endhighlight %}

The [`@Async`]({% post_url 2017-06-03-task-execution-and-scheduling-with-spring %}) annotation tells the Spring
Framework to run the methods of `EmailService` asynchrousnously when invoked.
The `html` of `sendHtmlEmail()` parameter contains the HTML markup. 

Next we need to implement the `ReportService`:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/jasper/mail/JasperReportsService.java' %}

{% highlight java %}
@Component
public class JasperReportsService implements ReportService {
  private final StorageService storageService;

  public JasperReportsService(StorageService storageService) {
    this.storageService = storageService;
  }

  @Override
  public String generateHtmlReport(String inputFileName, Map<String, Object> params) {
    return generateHtmlReport(inputFileName, params, new JREmptyDataSource());
  }

  @Override
  public String generateHtmlReport(String inputFileName, Map<String, Object> params,
    JRDataSource dataSource) {
    byte[] bytes = null;
    JasperReport jasperReport = null;
    try (ByteArrayOutputStream byteArray = new ByteArrayOutputStream()) {
      // Check if a compiled report exists
      if (storageService.jasperFileExists(inputFileName)) {
        jasperReport = (JasperReport) JRLoader
          .loadObject(storageService.loadJasperFile(inputFileName));
      }
      // Compile report from source and save
      else {
        String jrxml = storageService.loadJrxmlFile(inputFileName);
        jasperReport = JasperCompileManager.compileReport(jrxml);
        // Save compiled report. Compiled report is loaded next time
        JRSaver.saveObject(jasperReport,
          storageService.loadJasperFile(inputFileName));
      }
      JasperPrint jasperPrint = JasperFillManager.fillReport(jasperReport, params, dataSource);
      Exporter<ExporterInput, HtmlReportConfiguration, HtmlExporterConfiguration, HtmlExporterOutput> exporter;
      // HTML exporter
      exporter = new HtmlExporter();
      // Set output to byte array
      exporter.setExporterOutput(new SimpleHtmlExporterOutput(byteArray));
      // Set input source
      exporter.setExporterInput(new SimpleExporterInput(jasperPrint));
      // Export to HTML
      exporter.exportReport();
      bytes = byteArray.toByteArray();
    }
    catch (JRException | IOException e) {
      e.printStackTrace();
    }
    return new String(bytes);
  }

  // Implementation details omitted for brevity
}
{% endhighlight %}

Let's test sending a HTML mail. Be sure to set your SMTP configuration properties in `application.yaml`:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/jasper/Application.java' %}

{% highlight java %}
public class Application {
  private final ReportService reportService;
  private final EmailService emailService;
  private final ApplicationProperties properties;

  public Application(ReportService reportService, EmailService emailService, ApplicationProperties properties) {
    this.reportService = reportService;
    this.emailService = emailService;
    this.properties = properties;
  }
  // ...

  @Scheduled(cron = "${com.juliuskrah.cron}")
  void sendHTMLEmail() {
    Set<Recipient> recipients = properties.getMail().getRecipients();

    for (Recipient recipient : recipients) {
      Map<String, Object> params = new HashMap<>();
      params.put("username", recipient.getUsername());
      String html = reportService.generateHtmlReport("html", params);
      emailService.sendHtmlEmail(recipient.getEmail(), html);
    }
  }
}
{% endhighlight %}

The `html` String parameter is specified by 
({% include file-path.html file_path='src/main/resources/reports/html.jrxml' %}). Before you execute, do not forget
to set the values of `com.juliuskrah.cron` and `com.juliuskrah.mail.recipients[0].email` in the configuration file. 
Be sure to set an appropriate [cron](http://www.cronmaker.com/){:target="_blank"} 
[expression](http://www.quartz-scheduler.org/documentation/quartz-2.x/tutorials/crontrigger.html){:target="_blank"}.


![HTML without inline](https://i.imgur.com/PjLZBQc.png)

{:.image-caption}
*Jasper HTML Email*

Not very elegant, but you get the general idea.

# HTML Mail with Inline

Generating HTML in JasperReports with image resources is a bit tricky. JasperReports handles the image resources
separately before associating them to the HTML. 
We will implement the `generateInlineHtmlReport()` to handle the image resources.
The implementation of `generateInlineHtmlReport()` is similar to `generateHtmlReport()`. I will just highlight the
code change. You can view the full implementation in the linked file below:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/jasper/mail/JasperReportsService.java' %}

{% highlight java %}
public class JasperReportsService implements ReportService {
  // ...
  @Override
  public List<Object> generateInlineHtmlReport(String inputFileName,
    Map<String, Object> params, JRDataSource jRDataSource) {
    // ...
    // This will be populated with the image name, and byte[] resource
    Map<String, byte[]> resourcesMap = new HashMap<>();
    SimpleHtmlExporterOutput htmlExporterOutput = new SimpleHtmlExporterOutput(byteArray);
    htmlExporterOutput.setImageHandler(new MapHtmlResourceHandler((resourcesMap)) {
      @Override
      public String getResourcePath(String id) {
        // Add the Content ID
        return "cid:" + id;
      }
    });

    exporter.setExporterOutput(htmlExporterOutput);
    exporter.setExporterInput(new SimpleExporterInput(jasperPrint));
    exporter.exportReport();
    String html = new String(byteArray.toByteArray());
    result.add(html);
    // Add the populated map
    result.add(resourcesMap);
    return result;
  }
}
{% endhighlight %}

Implement the `sendHtmlEmail()` overload that can handle inline resources:


file: {% include file-path.html file_path='src/main/java/com/juliuskrah/jasper/mail/HtmlEmailService.java' %}

{% highlight java %}
public class HtmlEmailService implements EmailService {
  // ...
  @Override
  public void sendHtmlEmail(String recipient, String html, Map<String, byte[]> imageSource) {
    // Set true for inline
    final MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
    // ...

    // Set to true for HTML
    helper.setText(html, true);
    for(Entry<String, byte[]> val : imageSource.entrySet()) {
      helper.addInline(val.getKey(), new ByteArrayResource(val.getValue()), "image/png");
    }
    javaMail.send(message);
  }
}
{% endhighlight %}

Test the inline email:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/jasper/Application.java' %}

{% highlight java %}
public class Application {
  // ...

  @Scheduled(cron = "${com.juliuskrah.inline-cron}")
  void sendInlineHTMLEmail() {
    Set<Recipient> recipients = properties.getMail().getRecipients();

    for (Recipient recipient : recipients) {
      Map<String, Object> params = new HashMap<>();
      params.put("username", recipient.getUsername());
      List<Object> result = reportService.generateInlineHtmlReport("html_inline", params);
      String html = (String) result.get(0);
      Map<String, byte[]> imageSource = (Map<String, byte[]>) result.get(1);
      emailService.sendHtmlEmail(recipient.getEmail(), html, imageSource);
    }
  }
}
{% endhighlight %}

Set an appropriate cron expression for `com.juliuskrah.inline-cron` and that's all folks.


# Conclusion
In this post we learned how to generate HTML mail with JasperReports with and without inline images.      
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.

[JasperReports]:            https://community.jaspersoft.com/project/jasperreports-library
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org