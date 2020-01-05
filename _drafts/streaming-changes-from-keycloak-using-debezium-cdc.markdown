---
layout:     post
title:      Streaming Changes from Keycloak using Debezium (CDC)
categories: blog
tags:       java debezium micronaut cdc postgres
section:    blog
author:     juliuskrah
repo:       /keycloak-cdc/tree/master
---
> [Change data capture][Change Data Capture]{:target="_blank"} records insert, update, and delete    
  activity that is applied to a relational database table. 

# Introduction

In a microservices architecture, keeping consistent state between services is becomes tedious. Several
techniques have been employed over the years to tackle these challenges. One such approach is [Eventual Consistency][Eventual Consistency]{:target="_blank"}.
When you want to update an ElasticSearch index with data from PostgreSQL, you have to rely on one of these
techniques to keep data up-to-date. 

In this blog post I will discuss an alternate technique using Change Data Capture (CDC). CDC[^1] is a set 
of software design patterns used to determine the data that has changed so that action can be taken using 
the changed data. I have put together a little [demo application][demo app]{:target="_blank"} to showcase
this. In the demo I am streaming users created in [Keycloak][]{:target="_blank"} and saving them in a Java application.

# Debezium

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
psql> CREATE ROLE julius SUPERUSER LOGIN;
{% endhighlight %}

then configure the PostgreSQL server to allow replication;

file: *pg_hba.conf*

{% highlight conf %}
local   replication     julius                          trust   
host    replication     julius  127.0.0.1/32            trust   
host    replication     julius  ::1/128                 trust   
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

## Kafka Connect

[Eventual Consistency]:             https://docs.couchdb.org/en/stable/intro/consistency.html
[Change Data Capture]:              https://docs.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server?view=sql-server-ver15
[demo app]:                         https://{{ site.github.hostname }}/{{ site.github_username }}/{{ page.repo }}
[Keycloak]:                         https://www.keycloak.org/
[Debezium]:                         https://debezium.io/
[PostgreSQL]:                       https://jdbc.postgresql.org/documentation/head/replication.html
[Kafka]:                            https://kafka.apache.org/
[Zookeeper]:                        https://zookeeper.apache.org/
[docker]:                           https://{{ site.github.hostname }}/{{ site.github_username }}/{{ page.repo }}/docker-compose.nobuild.yml

### References

[^1]: https://en.wikipedia.org/wiki/Change_data_capture
[^2]: https://debezium.io/documentation/reference/0.10/connectors/index.html
[^3]: https://www.postgresql.org/docs/12/high-availability.html
[^4]: https://www.postgresql.org/docs/12/logical-replication-architecture.html