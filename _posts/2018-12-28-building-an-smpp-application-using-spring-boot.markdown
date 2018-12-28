---
layout:     post
title:      Building an SMPP Application using Spring Boot
categories: blog
tags:       java spring-boot maven
section:    blog
author:     juliuskrah
repo:       smpp/tree/master
---
> Short Message Peer-to-Peer (SMPP) is a protocol used by the telecommunications industry for exchanging 
  messages between Short Message Service Centers (SMSC) and/or External Short Messaging Entities (ESME).

# Introduction

In this post I will build a simple app that will send SMS using the SMPP protocol, bootstraped with Spring 
Boot. I will use the [Cloudhopper][Cloudhopper]{:target="_blank"} SMPP library for sending SMS.

SMPP is a level-7 TCP/IP protocol, which allows fast delivery of SMS messages. The most conmmonly used 
versions of SMPP are v3.3, the most widely supported standard, and v3.4, which adds transceiver support
(single connections that can send and receive messages).

# Prerequisites

- [Java Development Kit][JDK]{:target="_blank"}
- [Maven][]{:target="_blank"}
- SMPP account that supports v3.4

## Project Structure

At the end of this guide our folder structure will look similar to the following:

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__juliuskrah/
|  |  |  |  |  |__smpp/
|  |  |  |  |  |  |__Application.java
|  |  |  |  |  |  |__ApplicationProperties.java
|  |  |  |  |  |  |__ClientSmppSessionHandler.java
|  |  |  |  |  |  |__DeliveryReceipt.java
|  |  |__resources/
|  |  |  |__application.yaml
|__pom.xml
```

## Project Setup

Create a simple Spring Boot Application using any of the [bootstrap options][Install Spring Boot]{:target="_blank"}.

Add the following to your `pom.xml` after creating your Spring-Boot application:

{% highlight xml %}
<dependency>
  <groupId>com.fizzed</groupId>
  <artifactId>ch-smpp</artifactId>
  <version>5.0.9</version>
</dependency>
{% endhighlight %}

# Configuring the SMPP Session

In order to start using SMPP to send messages, you need to establish a session. The session is usually
short-lived; we will implement long-lived sessions later in this post by extending the session 
periodically.

To bind a session, we need a `SmppSessionConfiguration` and `SmppClient`. The `SmppSessionConfiguration`
class contains the configurable aspects of the SmppSession. In this class you can configure the:

- `Host`:      [smpp.serviceprovider.com]
- `Port`:      default 2775
- `System Id`: [username]
- `Password`:  [password]
- `BindType`:  TRANSCEIVER, TRANSMITTER or RECEIVER
- `Version`:   3.3, 3.4, 5.0

Configuring this class will look something like the following:

{% highlight java %}
public SmppSessionConfiguration sessionConfiguration() {
  SmppSessionConfiguration sessionConfig = new SmppSessionConfiguration();
  sessionConfig.setName("smpp.session");
  sessionConfig.setInterfaceVersion(SmppConstants.VERSION_3_4);
  sessionConfig.setType(SmppBindType.TRANSCEIVER);
  sessionConfig.setHost("<replace>");
  sessionConfig.setPort(2775);
  sessionConfig.setSystemId("<replace>");
  sessionConfig.setPassword("<replace>");
  sessionConfig.setSystemType(null);
  sessionConfig.getLoggingOptions().setLogBytes(false);
  sessionConfig.getLoggingOptions().setLogPdu(true);

  return sessionConfig;
}
{% endhighlight %}

> NOTE: Be sure to replace the values for `Host`, `SystemId` and `Password`.

I have set the bind-type as `TRANSCEIVER` because I want to be able to send SMS and receive delivery 
receipts.

What I have to do next is create the `SmppClient`:

{% highlight java %}
public SmppClient clientBootstrap() {
  return new DefaultSmppClient(Executors.newCachedThreadPool(), 2);
}
{% endhighlight %}

The `DefaultSmppClient` constructor takes an `ExecutorService` and expected number of sessions. In the
example above I am creating a CachedThreadPoolExecutor and assigning 2 concurrent sessions.

With these two in place I can now establish my `SmppSession`:

{% highlight java %}
@Bean(destroyMethod = "destroy")
public SmppSession session() throws SmppBindException, SmppTimeoutException, SmppChannelException,
    UnrecoverablePduException, InterruptedException {
  SmppSessionConfiguration config = sessionConfiguration();
  // Will use the DefaultSmppSessionHandler for now
  SmppSession session = clientBootstrap().bind(config, new DefaultSmppSessionHandler());

  return session;
}
{% endhighlight %}

In the above snippet, we are using a `DefaultSmppSessionHandler`. Later in this post I will create a 
custom `SmppSessionListener` that can handle delivery receipts.

At this point all necessary configuration is done. Everything wired up together until this point should
look like this:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/smpp/Application.java' %}

{% highlight java %}
@SpringBootApplication
public class Application {
  private static final Logger log = LoggerFactory.getLogger(Application.class);

  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }

  public SmppSessionConfiguration sessionConfiguration() {
    SmppSessionConfiguration sessionConfig = new SmppSessionConfiguration();
    sessionConfig.setName("smpp.session");
    sessionConfig.setInterfaceVersion(SmppConstants.VERSION_3_4);
    sessionConfig.setType(SmppBindType.TRANSCEIVER);
    sessionConfig.setHost("<replace>");
    sessionConfig.setPort(2775);
    sessionConfig.setSystemId("<replace>");
    sessionConfig.setPassword("<replace>");
    sessionConfig.setSystemType(null);
    sessionConfig.getLoggingOptions().setLogBytes(false);
    sessionConfig.getLoggingOptions().setLogPdu(true);

    return sessionConfig;
  }

  @Bean(destroyMethod = "destroy")
  public SmppSession session() throws SmppBindException, SmppTimeoutException, SmppChannelException,
      UnrecoverablePduException, InterruptedException {
    SmppSessionConfiguration config = sessionConfiguration();
    SmppSession session = clientBootstrap().bind(config, new DefaultSmppSessionHandler());

    return session;
  }

  public SmppClient clientBootstrap() {
    return new DefaultSmppClient(Executors.newCachedThreadPool(), 2);
  }
}
{% endhighlight %}


If you have setup your SMPP correctly with your service provider, run the code as a Spring-Boot app:

{% highlight posh %}
PS C:\> mvn spring-boot:run
{% endhighlight %}


## Send an SMS

Stop the aplication if it is still running. I will add a simple method that sends an SMS on
application startup:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/smpp/Application.java' %}

{% highlight java %}
// ...

private void sendTextMessage(SmppSession session, String sourceAddress, String message, String destinationAddress) {
  // Check if session is still active
  if (session.isBound()) {
    try {
      // request delivery
      boolean requestDlr = true;
      SubmitSm submit = new SubmitSm();
      byte[] textBytes;
      textBytes = CharsetUtil.encode(message, CharsetUtil.CHARSET_ISO_8859_1);
      // set encoding for sending SMS
      submit.setDataCoding(SmppConstants.DATA_CODING_LATIN1);
      if (requestDlr) {
        submit.setRegisteredDelivery(SmppConstants.REGISTERED_DELIVERY_SMSC_RECEIPT_REQUESTED);
      }

      if (textBytes != null && textBytes.length > 255) {
        submit.addOptionalParameter(
            new Tlv(SmppConstants.TAG_MESSAGE_PAYLOAD, textBytes, "message_payload"));
      } else {
        submit.setShortMessage(textBytes);
      }

      submit.setSourceAddress(new Address((byte) 0x05, (byte) 0x01, sourceAddress));
      submit.setDestAddress(new Address((byte) 0x01, (byte) 0x01, destinationAddress));
      // submit message to SMSC for delivery with a timeout of 10000ms
      SubmitSmResp submitResponse = session.submit(submit, 10000);
      if (submitResponse.getCommandStatus() == SmppConstants.STATUS_OK) {
        log.info("SMS submitted, message id {}", submitResponse.getMessageId());
      } else {
        throw new IllegalStateException(submitResponse.getResultMessage());
      }
    } catch (RecoverablePduException | UnrecoverablePduException | SmppTimeoutException | 
        SmppChannelException | InterruptedException e) {
        throw new IllegalStateException(e);
    }
    return;
  }
  throw new IllegalStateException("SMPP session is not connected");
}
{% endhighlight %}

Finally, let us call the method to send SMS:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/smpp/Application.java' %}

{% highlight java %}
// ...

public static void main(String[] args) {
  ConfigurableApplicationContext ctx = SpringApplication.run(Application.class, args);
  SmppSession session = ctx.getBean(SmppSession.class);
  new Application().sendTextMessage(session, "3299", "Hello World", "<replace with phone number>");
}
{% endhighlight %}

Add your phone number to `destinationAddress` and run the application. Remember to use the international
phone number format. 

At this point you are done and can choose not to follow the rest of this post.

## Making it Better

You may have noticed that, we hardcoded certain things like username, password and host in the source code.
This does not make the app very portable. Let's make use of externalized configuration, which has extensive
support in Spring-Boot.

I will create a class `ApplicationProperties` which contains all the external properties we need:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/smpp/ApplicationProperties.java' %}

{% highlight java %}
@ConfigurationProperties("sms")
public class ApplicationProperties {
  @NestedConfigurationProperty
  private final Async async = new Async();
  @NestedConfigurationProperty
  private final SMPP smpp = new SMPP();

  public Async getAsync() {
    return async;
  }

  public SMPP getSmpp() {
    return smpp;
  }

  public static class Async {
    /**
     * This number should be lower than the value assigned to the core-pool-size
     */
    private int smppSessionSize = 2;
    private int corePoolSize = 5;
    private int maxPoolSize = 50;
    private int queueCapacity = 10000;
    private int initialDelay = 1000;
    private int timeout = 10000;

    // omitted getters and setters
  }

  public static class SMPP {
    private String host;
    private String userId;
    private String password;
    private int port = 2775;
    private boolean requestDelivery = false;
    private boolean detectDlrByOpts = false;

    // omitted getters and setters
  }
}
{% endhighlight %}

I have added default values for most of the setings in the above class. I will override some of the 
defaults in a yaml file:

file: {% include file-path.html file_path='src/main/resources/application.yaml' %}

{% highlight yaml %}
sms:
  smpp:
    host: <replace>
    user-id: <replace>
    password: <replace>
    requestDelivery: true
{% endhighlight %}

In the above, I am overriding the default value of `requestDelivery` to `true`.

In order to use this in our simple application, I need to register the `ApplicationProperties`
configuration class:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/smpp/Application.java' %}

{% highlight java %}
@SpringBootApplication
@EnableConfigurationProperties(ApplicationProperties.class)
public class Application {
  // ...
}
{% endhighlight %}

I will rewrite `sessionConfiguration` method to use the externalized configuration:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/smpp/Application.java' %}

{% highlight java %}
// ...

public SmppSessionConfiguration sessionConfiguration(ApplicationProperties properties) {
SmppSessionConfiguration sessionConfig = new SmppSessionConfiguration();
  sessionConfig.setName("smpp.session");
  sessionConfig.setInterfaceVersion(SmppConstants.VERSION_3_4);
  sessionConfig.setType(SmppBindType.TRANSCEIVER);
  sessionConfig.setHost(properties.getSmpp().getHost());
  sessionConfig.setPort(properties.getSmpp().getPort());
  sessionConfig.setSystemId(properties.getSmpp().getUserId());
  sessionConfig.setPassword(properties.getSmpp().getPassword());
  sessionConfig.setSystemType(null);
  sessionConfig.getLoggingOptions().setLogBytes(false);
  sessionConfig.getLoggingOptions().setLogPdu(true);

  return sessionConfig;
}
{% endhighlight %}

At the beginning of this post I talked about receiving delivery receipts. The default implementation of
`SmppSessionListener` is to discard received PDUs. So I will extend the default implementation class and
handle delivery receipts in there:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/smpp/ClientSmppSessionHandler.java' %}

{% highlight java %}
public class ClientSmppSessionHandler extends DefaultSmppSessionHandler {
  private static final Logger log = LoggerFactory.getLogger(ClientSmppSessionHandler.class);
  private final ApplicationProperties properties;

  public ClientSmppSessionHandler(ApplicationProperties properties) {
    this.properties = properties;
  }

  private String mapDataCodingToCharset(byte dataCoding) {
    // implementaion in the github repository
  }

  @Override
  @SuppressWarnings("rawtypes")
  public PduResponse firePduRequestReceived(PduRequest request) {
    PduResponse response = null;
    try {
      if (request instanceof DeliverSm) {
        String sourceAddress = ((DeliverSm) request).getSourceAddress().getAddress();
        String message = CharsetUtil.decode(((DeliverSm) request).getShortMessage(),
            mapDataCodingToCharset(((DeliverSm) request).getDataCoding()));
        log.debug("SMS Message Received: {}, Source Address: {}", message.trim(), sourceAddress);

        boolean isDeliveryReceipt = false;
        if (properties.getSmpp().isDetectDlrByOpts()) {
          isDeliveryReceipt = request.getOptionalParameters() != null;
        } else {
          isDeliveryReceipt = SmppUtil.isMessageTypeAnyDeliveryReceipt(
            ((DeliverSm) request).getEsmClass());
        }

        if (isDeliveryReceipt) {
          DeliveryReceipt dlr = DeliveryReceipt.parseShortMessage(message, ZoneOffset.UTC);
          // logging delivery here, but you can do something more useful over here
          log.info("Received delivery from {} at {} with message-id {} and status {}", sourceAddress,
              dlr.getDoneDate(), dlr.getMessageId(), DeliveryReceipt.toStateText(dlr.getState()));
        }
      }
      response = request.createResponse();
    } catch (Throwable error) {
      log.warn("Error while handling delivery", error);
      response = request.createResponse();
      response.setResultMessage(error.getMessage());
      response.setCommandStatus(SmppConstants.STATUS_UNKNOWNERR);
    }
    return response;
  }
}
{% endhighlight %}

The `DeliveryReceipt` class is a utility class I created to extract delivery details from `PduRequest`. 
I will not paste the contents of the class here (it is quite lengthy), you can view its contents in the 
[Github repository](https://github.com/juliuskrah/smpp/tree/master/src/main/java/com/juliuskrah/smpp/DeliveryReceipt.java).


We just need to register this handler to take advantage of the delivery receipt handling:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/smpp/Application.java' %}

{% highlight java %}
@Bean(destroyMethod = "destroy")
public SmppSession session(ApplicationProperties properties) throws SmppBindException, 
    SmppTimeoutException, SmppChannelException,
    UnrecoverablePduException, InterruptedException {
  SmppSessionConfiguration config = sessionConfiguration(properties);
  SmppSession session = clientBootstrap().bind(config, new ClientSmppSessionHandler(properties));

  return session;
}
{% endhighlight %}

If you run the application now, you will see delivery receipts printed in the console. This solution 
however is far from ideal. As I said earlier, the Sessions are short-lived; given a scenario where
a mobile device is switched off, the SMS will be delivered only when the device is switched back on.
In this same scenario, the SMPP Session may be closed when the delivery is received, leading to lost
delivery receipts. To overcome this limitation I have to design the application to periodically refresh
the session before it un-binds.

I will use `EnquireLinkResp` to periodically extend the session:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/smpp/Application.java' %}

{% highlight java %}
@EnableScheduling
@SpringBootApplication
@EnableConfigurationProperties(ApplicationProperties.class)
public class Application {
  // ...

  @Scheduled(
      initialDelayString = "${sms.async.initial-delay}", 
      fixedDelayString = "${sms.async.initial-delay}")
  void enquireLinkJob() {
    if (session.isBound()) {
      try {
        log.info("sending enquire_link");
        EnquireLinkResp enquireLinkResp = session.enquireLink(new EnquireLink(),
            properties.getAsync().getTimeout());
        log.info("enquire_link_resp: {}", enquireLinkResp);
      } catch (SmppTimeoutException e) {
        log.info("Enquire link failed, executing reconnect; " + e);
        log.error("", e);
      } catch (SmppChannelException e) {
        log.info("Enquire link failed, executing reconnect; " + e);
        log.warn("", e);
      } catch (InterruptedException e) {
        log.info("Enquire link interrupted, probably killed by reconnecting");
      } catch (Exception e) {
        log.error("Enquire link failed, executing reconnect", e);
      }
    } else {
      log.error("enquire link running while session is not connected");
    }
  }
}
{% endhighlight %}

I have added annotation on top of the class to `@EnableScheduling` which allows spring to process
the `@Scheduled` annotation. The fixedDelayString value `${sms.async.initial-delay}` is set in the 
`application.yaml` file:

file: {% include file-path.html file_path='src/main/resources/application.yaml' %}

{% highlight yaml %}
sms:
  async:
    initial-delay: 30000
{% endhighlight %}

This will cause the application to enquire link every 30 seconds. This solution is far from perfect. When 
internet connectivity is lost on the server running the application, this will not work. Also when the 
Session unexpectedly unbinds this solution will not work. I have implemented a more robust solution
on the `persistent` branch of the Github repository accompanying this post.

Before we wrap up, it is not very good practice to store sentive information in plain text on files. There 
are several best practices and solutions out there e.g. using Vault from Hashicorp. I am going to implement
the simplest solution by using environment variables:

{% highlight posh %}
PS C:\> set SMS_SMPP_HOST=<host>
PS C:\> set SMS_SMPP_USER_ID=<user>
PS C:\> set SMS_SMPP_PASSWORD=<password>
{% endhighlight %}

# Conclusion

In this post we looked briefly at the SMPP protocol and its usage within the JVM. We also covered how to 
use Spring Boot to make developing an SMPP client easier. 

As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep
doing cool things :+1:.


[Install Spring Boot]:       https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#getting-started-installing-spring-boot
[Cloudhopper]:              https://github.com/fizzed/cloudhopper-smpp
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Maven]:                    http://maven.apache.org