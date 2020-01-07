---
layout:     post
title:      Streaming Changes from Keycloak using Debezium (CDC)
date:       2020-01-06 15:59:40 +0000
categories: blog
tags:       java debezium micronaut cdc postgres
section:    blog
author:     juliuskrah
repo:       /keycloak-cdc/tree/master
---
> [Change data capture][Change Data Capture]{:target="_blank"} records insert, update, and delete
  activity that is applied to a relational database table. 

# Introduction

In a microservices architecture, keeping consistent state between services becomes tedious. Several
techniques have been employed over the years to tackle these challenges. One such approach is [Eventual Consistency][Eventual Consistency]{:target="_blank"}.
When you want to update an ElasticSearch index with data from PostgreSQL, you have to rely on one of these
techniques to keep data up-to-date. 

In this blog post I will discuss an alternate technique using Change Data Capture (CDC). CDC[^1] is a set 
of software design patterns used to determine the data that has changed so that action can be taken using 
the changed data. I have put together a little [demo application][demo app]{:target="_blank"} to showcase
this. In the demo I am streaming users created in [Keycloak][]{:target="_blank"} and saving them in a Java application.

# Solution Architecture

![CDC Architecture](https://i.imgur.com/DmfMs8o.png)
{:.image-caption}
Fig 1 *Solution showing all the components functioning together*

There are several open source CDC tools out there in the wild. I have chosen to go with [Debezium][]{:target="_blank"}
and [PostgreSQL][]{:target="_blank"}; you can choose any combination of your choice. To start debzium, 
for CDC the following ritual must be performed.

First you have to configure your database of choice to enable CDC streaming. At the time of this writing
debezium[^2] supports the following databases:

- MongoDB
- MySQL
- [PostgreSQL][]{:target="_blank"}
- SQL Server
- Oracle
- Cassandra

## PostgreSQL

We are using PostgreSQL for this demo. PostgreSQL uses a _logical decoder_ that writes change events to
a Write Ahead Log (WAL)[^3] after they are committed. 
PostgreSQL's _logical decoding_ feature was first introduced in version 9.4
and is a mechanism which allows the extraction of the changes which were committed to the transaction log.
The changes are processed in a user-friendly manner via the help of an _output plugin_. The logical 
decoding output plugin must be installed prior to running the PostgreSQL server and enabled together with
a replication slot. The output plugins supported by debezium are:

- [decoderbufs](https://github.com/debezium/postgres-decoderbufs){:target="_blank"}
- [wal2json](https://github.com/eulerto/wal2json){:target="_blank"}
- pgoutput[^4] - standard logical decoding plug-in in PostgreSQL 10+

I will be using **pgoutput** because it requires no additional libraries to be installed. Next enable
a replication slot, and configure a user with sufficient privileges to perform the replication.

According to the postgres documentation, a user with REPLICATION role is enough for CDC, however in my
case I've never managed to get it to work with the debezium postgres connector. Let me know in the 
comments if you are successful. I create a user with SUPERUSER role for this purpose;

{% highlight bash %}
psql> CREATE ROLE postgres SUPERUSER LOGIN;
psql> ALTER ROLE postgres WITH PASSWORD 'postgrespwd';
{% endhighlight %}

then configure the PostgreSQL server to allow replication;

file: *pg_hba.conf*

{% highlight conf %}
local   replication     postgres                          trust   
host    replication     postgres  127.0.0.1/32            trust   
host    replication     postgres  ::1/128                 trust   
{% endhighlight %}

and finally configure the replication slot:

file: _postgresql.conf_

{% highlight conf %}
wal_level = logical             
max_wal_senders = 1             
max_replication_slots = 5      
{% endhighlight %}

With PostgreSQL up and running, create the database and user for Keycloak:

{% highlight conf %}
pasql> CREATE USER sample WITH PASSWORD 'sample';
pasql> CREATE DATABASE sample;
pasql> GRANT ALL PRIVILEGES ON DATABASE sample TO sample;
pasql> \c sample;
pasql> CREATE SCHEMA IF NOT EXISTS AUTHORIZATION sample;
{% endhighlight %}

## Kafka and Zookeeper

You now need to configure [Kafka][]{:target="_blank"} and [Zookeeper][]{:target="_blank"}. I will not 
demonstrate how to setup Kafka and Zookeeper; everything required to run this demo is setup in a 
[docker][]{:target="_blank"} file.

## Keycloak

Once keycloak is installed, import the [sample realm][]{:target="_blank"}. You can also run the 
[compose file][docker]{:target="_blank"} to get everything up and running.

## Kafka Connect

Debezium is a [Kafka Connect][connect]{:target="_blank"} connector; It can be easily setup using the
[compose file][docker]{:target="_blank"}.

Once the connector is ready to recieve connections, we make a post request to create our CDC Connector:

{% highlight bash linenos %}
> curl -s \
     -X "POST" "http://localhost:8083/connectors/" \
     -H "Content-Type: application/json" \
     -d '{
  "name": "keycloak-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "slot.name": "keycloak_debezium",
    "plugin.name": "pgoutput",
    "database.user": "postgres",
    "database.password": "postgrespwd",
    "database.dbname": "sample",
    "database.server.name": "keycloak",
    "schema.whitelist": "sample",
    "table.whitelist": "sample.user_role_mapping,sample.user_group_membership,sample.keycloak_role,sample.user_entity"
  }
}'
{% endhighlight%}

A brief description of the payload above; 

- line 5 is the name for this CDC connector. It can be whatever you choose
- line 8 is the database hostname, it can also take an IP address
- line 10 indicates the replication slot you want to create in PostgreSQL server
- on line 11 I am using the _pgoutput_ plugin. Other alternatives are _decoderbufs_ and _wal2json_, to use
  these alternatives, you have to install the plugins
- on line 15 I specify a server name (Can be any name you choose). This name is used by the connector
  when creating the kafka topics. More on this in the next section.
- lines 16 and 17 I indicate which tables I want to capture changes on.

You can read more on the debezium documentation.

The debezium connector will now recieve changes from the PostgreSQL WAL and publish them to Kafka.

I created a sink application to process the changes from Kafka.

## CDC Application

I created a representation of the CDC payload from Kafka:

file: {% include file-path.html file_path='app-service/src/main/java/com/juliuskrah/cdc/dto/KeycloakCdcDto.java' %}

{% highlight java %}
public class KeycloakCdcDto<T> {
    private Map<String, Object> schema;
    private Payload<T> payload;

    // Getters and Setters omitted for brevity

    public static class Payload<T> {
        private T before;
        private T after;
        private Map<String, Object> source;
        private char op;
        @JsonProperty("ts_ms")
        private Instant tsMs;
        
        // Getters and Setters omitted for brevity
    }
}
{% endhighlight %}

As already hinted in the previous section debezium creates topics in kafka for each table it captures
events on in the following format `<database.server.name>.<schema_name>.<table_name>`. In our example
above, the changes on the `user_entity` table will be stored in the `keycloak.sample.user_entity` topic.

And the consumer for the event:

file: {% include file-path.html file_path='app-service/src/main/java/com/juliuskrah/cdc/sink/KafkaConsumer.java' %}

{% highlight java %}
@KafkaListener(groupId = "${app.group.id:group-id}", offsetReset = OffsetReset.EARLIEST)
public class KafkaConsumer {

    @Topic("keycloak.sample.user_entity")
    void onReceiveUser(@KafkaKey String key, @Nullable String event) {
        // process here
    }
}
{% endhighlight %}

The full source code is {% include source.html %}.

# Conclusion

Integration between microservices in any deployment is crucial, and in this post I have discussed using
CDC to tackle it. We also tackled setting up PostgreSQL for replication, glanced over setting up Keycloak,
Zookeeper and Kafka. We looked at setting up the debezium Kafka Connector and a sample application to 
consume the CDC event stream. Until the next post, keep doing cool things :+1:.

[Eventual Consistency]:             https://docs.couchdb.org/en/stable/intro/consistency.html
[Change Data Capture]:              https://docs.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server?view=sql-server-ver15
[demo app]:                         https://{{ site.github.hostname }}/{{ site.github_username }}/{{ page.repo }}
[Keycloak]:                         https://www.keycloak.org/
[Debezium]:                         https://debezium.io/
[PostgreSQL]:                       https://jdbc.postgresql.org/documentation/head/replication.html
[Kafka]:                            https://kafka.apache.org/
[Zookeeper]:                        https://zookeeper.apache.org/
[docker]:                           https://{{ site.github.hostname }}/{{ site.github_username }}/{{ page.repo }}/docker-compose.nobuild.yml
[sample realm]:                     https://raw.githubusercontent.com/juliuskrah/keycloak-cdc/master/keycloak/sample_realm.json
[Connect]:                          https://kafka.apache.org/documentation/#connect

### References

[^1]: https://en.wikipedia.org/wiki/Change_data_capture
[^2]: https://debezium.io/documentation/reference/0.10/connectors/index.html
[^3]: https://www.postgresql.org/docs/12/high-availability.html
[^4]: https://www.postgresql.org/docs/12/logical-replication-architecture.html