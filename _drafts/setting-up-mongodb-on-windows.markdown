---
layout:     post
title:      Setting up MongoDB on Windows
categories: blog
tags:       mongoDB Windows
section:    blog
author:     Julius Krah
---
> [MongoDB][]{:target="_blank"} is an open-source, document database designed for ease of development and scaling.

# Introduction
In this blog post, we are going to learn how to setup mongoDB on a Windows machine. At the end of this post, we would have 
accomplished the following objectives:

- Setup and configure mongoDB as a Windows service
- Launch and interact with the [mongo][]{:target="_blank"} Shell JavaScript Interface
- Write and retrieve [BSON][]{:target="_blank"} documents to and from mongoDB  

In the next section of this post, we are going to look at how to install mongoDB on a windows machine after it has been downloaded.
The section that follows after that will look at how to launch the interactive shell after server is configured as a windows 
service. The final section will then cover the basics of the BSON interchange format and how to save and retrieve it to and from
mongoDB.

# Setting up MongoDB
Before you install the `.msi` package, make sure your system meets the following requirements:

- Windows Server 2008 R2
- Windows Vista or later

To know which version of Windows you are running, enter the following command in **Command Prompt** or **Powershell**:

{% highlight posh %}
wmic os get caption
{% endhighlight %}

Once you machine meets the above requirements, you have to determine which build of MongoDB you need (32 bit or 64 bit). 
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
    Press the `Win` key, type `cmd.exe`, and press `Ctrl + Shift + Enter` to run the **Command Prompt** as Administrator.  
    Execute the remaining steps from the Administrator command prompt.  
2.  `Install MongoDB for Windows.`  
    Change to the directory containing the `.msi` installation binary of your choice and invoke:

    {% highlight posh %}
    msiexec.exe /q /i mongodb-win32-x86_64-2008plus-ssl-3.2.11-signed.msi ^
            INSTALLLOCATION="C:\Program Files\MongoDB\Server\3.2.11\" ^
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
            INSTALLLOCATION="C:\Program Files\MongoDB\Server\3.2.11\" ^
            ADDLOCAL="MonitoringTools,ImportExportTools,MiscellaneousTools"
{% endhighlight %}

## Configure a Windows Service for MongoDB Community Edition
Open an Administrative **Command Prompt** by pressing the `Win` key. Type `cmd.exe`, and press `Ctrl + Shift + Enter` to open
**Command Prompt** in elevated mode.  
Execute the remaining steps from the Administrator command prompt.

Create directories for your database and log files:

{% highlight posh %}
mkdir c:\data\db
mkdir c:\data\log
{% endhighlight %}

Execute the above commands one after the other.

Create a configuration file (`mongod.cfg`). The file **must** set [`systemLog.path`](https://docs.mongodb.com/manual/reference/configuration-options/#systemLog.path){:target="_blank"}.
For example, create a file at `C:\Program Files\MongoDB\Server\3.2\mongod.cfg` that specifies both [`systemLog.path`](https://docs.mongodb.com/manual/reference/configuration-options/#systemLog.path){:target="_blank"} 
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
If you are following along with this blog and you used all the defaults, this value will be `C:\Program Files\MongoDB\Server\3.2`.  
Next we would add the `MONGO_HOME` variable to the `path`. Select `path` from _System Variables_ and click **Edit...** add `%MONGO_HOME%\bin`
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
The [mongo][]{:target="_blank"} shell is an interactive JavaScript interface to MongoDB. You can use the 
[mongo][]{:target="_blank"} shell to query and update data as well as perform administrative operations.

> **NOTE**  
  The rest of this article assumes that you have MongoDB running as a service on Windows and added MongoDB to
  the `path`

To start the `mongo` shell and connect to your MongoDB instance running on **localhost** with **default port**:

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

To check the current database you are connected to run:

{% highlight posh %}
> db
{% endhighlight %}

The interactive shell will output something like:

{% highlight posh %}
test
{% endhighlight %}

[MongoDB]: https://www.mongodb.com/what-is-mongodb
[mongo]: https://docs.mongodb.com/manual/mongo/
[BSON]: http://bsonspec.org/