---
layout:     post
title:      'Setting up MongoDB on Windows'
date:       2016-11-26 22:42:17 +0000
categories: blog
tags:       mongodb windows
section:    blog
author:     Julius Krah
---
> [MongoDB][]{:target="_blank"} is an open-source, document database designed for ease of development and scaling.

# Introduction
In this blog post, we are going to learn how to setup mongoDB on Windows. At the end of this post, we would have 
accomplished the following objectives:

- Setup and configured mongoDB as a Windows service
- Launched and interacted with the [`mongo`][mongo]{:target="_blank"} Shell JavaScript Interface
- Written and retrieved [BSON][]{:target="_blank"} documents to and from mongoDB using the `mongo` Shell 

In the next section of this post, we are going to look at how to install mongoDB on Windows.
The section that follows will show you how to launch the interactive shell after server is configured as a Windows 
service. The penultimate section will then cover the basics of the BSON interchange format and how to save and retrieve documents
to and from mongoDB.

# Setting up MongoDB
Before you install the `.msi` package, make sure your system meets the following requirements:

- Windows Server 2008 R2
- Windows Vista or later

To know which version of Windows you are running, enter the following command in **Command Prompt** or **Powershell**:

{% highlight posh %}
wmic os get caption
{% endhighlight %}

Once your machine meets the above requirements, you have to determine which build of MongoDB you need (32 bit or 64 bit). 
To know your Windows OS architecture, run the following command in **Command Prompt** or **Powershell**:

{% highlight posh %}
wmic os get osarchitecture
{% endhighlight %}

To setup mongoDB on Windows, first download the latest `.msi` package from the [MongoDB downloads center](https://www.mongodb.com/download-center#community){:target="_blank"}.
Ensure you download the correct version of MongoDB for your Windows system. The 64-bit versions of MongoDB do not work 
with 32-bit Windows.

## Install MongoDB Community Edition
To install MongoDB, you have two options:

- Interactive Installation
- Unattended Installation

### Interactive Installation
In Windows Explorer, locate the downloaded MongoDB `.msi` file, which typically is located in the default Downloads folder.
Double-click the `.msi` file. A set of screens will appear to guide you through the installation process.
You may specify an installation directory if you choose the `Custom` installation option.

> **Note**  
  These instructions assume that you have installed MongoDB to `C:\Program Files\MongoDB\Server\3.2\`.

MongoDB is self-contained and does not have any other system dependencies. You can run MongoDB from any folder you choose. 
You may install MongoDB in any folder (e.g. `D:\test\mongodb`).

### Unattended Installation
You may install MongoDB Community unattended on Windows from the command line using `msiexec.exe`.

1.  `Open an Administrator command prompt.`  
    Press the `Win`{:.fa .fa-windows} key, type `cmd.exe`, and press `Ctrl + Shift + Enter` to run the **Command Prompt** as Administrator.  
    Execute the remaining steps from the Administrator command prompt.  
2.  `Install MongoDB for Windows.`  
    Change to the directory containing the `.msi` installation binary of your choice and invoke:

    {% highlight posh %}
    msiexec.exe /q /i mongodb-win32-x86_64-2008plus-ssl-3.2.11-signed.msi ^
            INSTALLLOCATION="C:\Program Files\MongoDB\Server\3.2\" ^
            ADDLOCAL="all"
    {% endhighlight %}

You can specify the installation location for the executable by modifying the **INSTALLLOCATION** value.  
By default, this method installs all MongoDB binaries. To install specific MongoDB component sets, you can specify 
them in the **ADDLOCAL** argument using a comma-separated list including one or more of the following component sets:

|Component Set       | Binaries                                                                   |
|--------------------|:---------------------------------------------------------------------------|
|`Server`            |`mongod.exe`                                                                |
|`Router`            |`mongos.exe`                                                                |
|`Client`            |`mongo.exe`                                                                 |
|`MonitoringTools`   |`mongostat.exe`, `mongotop.exe`                                             |
|`ImportExportTools` |`mongodump.exe`, `mongorestore.exe`, `mongoexport.exe`, `mongoimport.exe`   |
|`MiscellaneousTools`|`bsondump.exe`, `mongofiles.exe`, `mongooplog.exe`, `mongoperf.exe`         |
{:.comparison}

For instance, to install only the MongoDB utilities, invoke:

{% highlight posh %}
msiexec.exe /q /i mongodb-win32-x86_64-2008plus-ssl-3.2.11-signed.msi ^
            INSTALLLOCATION="C:\Program Files\MongoDB\Server\3.2\" ^
            ADDLOCAL="MonitoringTools,ImportExportTools,MiscellaneousTools"
{% endhighlight %}

## Configure a Windows Service for MongoDB Community Edition
Open an Administrative **Command Prompt** by pressing the `Win`{:.fa .fa-windows} key. Type `cmd.exe`, and press `Ctrl + Shift + Enter` to open
**Command Prompt** in elevated mode.  
Execute the remaining steps from the Administrator command prompt.

Create directories for your database and log files:

{% highlight posh %}
mkdir c:\data\db
mkdir c:\data\log
{% endhighlight %}

Execute the above commands one after the other.

Create a configuration file (`mongod.cfg`). The file **must** set [`systemLog.path`](https://docs.mongodb.com/manual/reference/configuration-options/#systemLog.path){:target="_blank"}.
For example, create a file at `C:\Program Files\MongoDB\Server\3.2\mongod.cfg` or whatever location you prefer that specifies both 
[`systemLog.path`](https://docs.mongodb.com/manual/reference/configuration-options/#systemLog.path){:target="_blank"} 
and [`storage.dbPath`](https://docs.mongodb.com/manual/reference/configuration-options/#storage.dbPath){:target="_blank"}:

{% highlight yaml %}
systemLog:
    destination: file
    path: c:\data\log\mongod.log
storage:
    dbPath: c:\data\db
{% endhighlight %}


`systemLog.path`: This is the location of your logs  
`storage.dbPath`: This is where you data is stored

> **Important**  
  Run all of the following commands in Command Prompt with “Administrative Privileges”.

Before we go any further, we will add MongoDB to the `path`. Please refer to [this page](https://www.java.com/en/download/help/path.xml){:target="_blank"}
if you are not familiar with creating environment variables on Windows.  
Add a new _System Environment Variable_. Set the _Variable name_ to `MONGO_HOME`; and the _Variable value_ to `path\to\mongo\root`.
If you are following along this blog and you used all the defaults, this value will be `C:\Program Files\MongoDB\Server\3.2`.  
Next we would add the `MONGO_HOME` variable to the `path`. Select `path` from _System Variables_ and click **Edit...** add `;%MONGO_HOME%\bin`
at the end of the `path` _Variable value_.

Install the MongoDB service by starting `mongod.exe` with the `--install` option and the `-config` option to specify the previously 
created configuration file:

{% highlight posh %}
mongod --config "C:\Program Files\MongoDB\Server\3.2\mongod.cfg" --install
{% endhighlight %}

To use an alternate `dbpath`, specify the path in the configuration file (e.g. `C:\Program Files\MongoDB\Server\3.2\mongod.cfg`) or 
on the command line with the `--dbpath` option.

To start the MongoDB service use the following command:

{% highlight posh %}
net start MongoDB
{% endhighlight %}

To stop the MongoDB service use the following command:

{% highlight posh %}
net stop MongoDB
{% endhighlight %}

To remove the MongoDB service use the following command:

{% highlight posh %}
mongod --remove
{% endhighlight %}


# The mongo Shell
Now that we have MongoDB set up as a Windows service, we can start hacking with the `mongo` shell.  
The [`mongo`][mongo]{:target="_blank"} shell is an interactive JavaScript interface to MongoDB. You can use the 
[`mongo`][mongo]{:target="_blank"} shell to query and update data as well as perform administrative operations.

> **NOTE**  
  The rest of this article assumes that you have MongoDB running as a service on Windows and MongoDB is on
  the `path`

In this section we will be learning basic hacks with the `mongo` Shell. In the next section we will look at inserting into and
retrieving documents from the database using the `mongo` Shell. To start the `mongo` shell and connect to your MongoDB instance 
running on **localhost** with **default port**:

{% highlight posh %}
mongo
{% endhighlight %}

The command above will generate an output similar to this:

{% highlight posh%}
MongoDB shell version: 3.2.11
connecting to: test
{% endhighlight %}

When starting, `mongo` checks the user’s `HOME` directory for a JavaScript file named `.mongorc.js`. 
If found, `mongo` interprets the content of `.mongorc.js` before displaying the prompt for the first time. 
If you use the shell to evaluate a JavaScript file or expression, either by using the `--eval` option on the command line 
or by specifying a `.js` file to `mongo`, `mongo` will read the `.mongorc.js` file _after_ the JavaScript has 
finished processing. You can prevent `.mongorc.js` from being loaded by using the `--norc` option.

To display the database you are using, type `db`:

{% highlight posh %}
> db
{% endhighlight %}

The operation should return the default database `test`:

{% highlight posh %}
test
{% endhighlight %}

You should also note that the `mongo` Shell is a JavaScript interface. You can write regular JavaScript in the Shell:

{% highlight posh %}
> var intValue = 0;
{% endhighlight %}

We can then write a regular `while loop` and then print the incrementing value of `intValue`:

{% highlight posh %}
> while (intValue < 10) {
... intValue++;
... print(intValue);
... }
{% endhighlight %}

From the example above, there are a few things to note. If you end a line with an open parenthesis ('`(`'), an open 
brace ('`{`'), or an open bracket ('`[`'), then the subsequent lines start with ellipsis ("`...`") until you enter 
the corresponding closing parenthesis ('`)`'), the closing brace ('`}`') or the closing bracket ('`]`'). The `mongo` 
shell waits for the closing parenthesis, closing brace, or the closing bracket before evaluating the code.

With this in mind the output of the previously executed JavaScript loop will be:

{% highlight posh %}
1
2
3
4
5
6
7
8
9
10
{% endhighlight %}

We will look at `BSON` documents in the next section.

# Write and Retrieve BSON Documents
A record in MongoDB is a document, which is a data structure composed of field and value pairs. MongoDB documents are similar to 
`JSON` objects. The values of fields may include other documents, arrays, and arrays of documents.

`BSON`, short for Bin­ary JSON, is a bin­ary-en­coded seri­al­iz­a­tion of JSON-like doc­u­ments. Like JSON, BSON sup­ports the em­bed­ding of 
doc­u­ments and ar­rays with­in oth­er doc­u­ments and ar­rays. BSON also con­tains ex­ten­sions that al­low rep­res­ent­a­tion of data types that 
are not part of the JSON spec. For ex­ample, BSON has a `Date type` and a `BinData type`.

A document in MongoDB will be represented similar to this:

{% highlight json %}
{
    name: "Julius",
    age: 26,
    status: "Alive",
    groups: ["google", "github"]
}
{% endhighlight %}

The advantages of using documents are:

- Documents (i.e. objects) correspond to native data types in many programming languages.
- Embedded documents and arrays reduce need for expensive joins.
- Dynamic schema supports fluent polymorphism.

In this section we are going to write into and read from a MongoDB database. To demonstrate this, we will be using the `test` database.
Launch the `mongo` Shell interface and execute the following commands to switch to the `test` database:

{% highlight posh %}
> use test
{% endhighlight %}

MongoDB stores BSON documents, i.e. data records, in `collections`; the `collections` in databases. MongoDB stores documents in 
`collections`. `Collections` are analogous to tables in relational databases.

Let us write our first document into MongoDB. To write documents into MongoDB we use Create Operations.  
Create or insert operations add new documents to a collection. If the collection does not currently exist, insert operations 
will create the collection.

MongoDB provides the following methods to insert documents into a collection:

- db.collection.insert()
- db.collection.insertOne()
- db.collection.insertMany()

For this post we will be using only the `insert()` method. In MongoDB, insert operations target a single collection. 
All write operations in MongoDB are atomic on the level of a single document:

{% highlight posh %}
> db.users.insert(
  {
    name: "Julius",
    age: 26,
    status: "Alive"
  }
)
{% endhighlight %}

This will create the `users` collection and give the sample output:

{% highlight posh %}
WriteResult({ "nInserted" : 1 })
{% endhighlight %}

MongoDB uses Read operations to retrieve documents from a collection; i.e. queries a collection for documents. MongoDB provides the 
following methods to read documents from a collection:

- db.collection.find()
- db.collection.findOne()

We will retreive the document we just inserted into the `users` collection with the `find()` method:

{% highlight posh %}
> db.users.find()
{% endhighlight %}

You will get output similar to this:

{% highlight json %}
{ "_id" : ObjectId("5839f9585165b6512261035c"), "name" : "Julius", "age" : 26, "status" : "Alive" }
{% endhighlight %}

From the above output there is one thing of note here `_id`. In MongoDB, each document stored in a collection requires a 
unique `_id` field that acts as a primary key. If an inserted document omits the `_id` field, the MongoDB driver automatically 
generates an [ObjectId][]{:target="_blank"} for the `_id` field.  
The above output can be made pretty with indentation by calling `pretty()` on the returned cursor of `find()`:

{% highlight posh %}
> db.users.find().pretty()
{% endhighlight %}

# Conclusion
In this blog post, we learned how to setup MongoDB as a Windows service. We also learned how to use and interact with the `mongo` 
JavaScript Shell interface. We finished up with working with BSON documents.  
I enjoyed writing this post, I hope you did too and until the next post, keep doing cool things :smile:.


[MongoDB]: https://www.mongodb.com/what-is-mongodb
[mongo]: https://docs.mongodb.com/manual/mongo/
[BSON]: http://bsonspec.org/
[ObjectId]: https://docs.mongodb.com/v3.2/reference/method/ObjectId/