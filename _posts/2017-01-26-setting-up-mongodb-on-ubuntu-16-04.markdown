---
layout:     post
title:      'Setting up MongoDB on Ubuntu 16.04'
date:       2017-01-26 13:24:21 +0000
categories: blog
tags:       mongodb ubuntu
section:    blog
author:     Julius Krah
---
> [MongoDB][]{:target="_blank"} is an open-source, document database designed for ease of development and scaling.

# Introduction
In this post we will learn how to setup MongoDB on [Ubuntu Server][Ubuntu]{:target="_blank"}. If you are looking for instructions on
[how to setup MongoDB on Windows]({% post_url 2016-11-26-setting-up-mongodb-on-windows %}) check out my previous post on the topic.
At the end of this post, we would have accomplished the following objectives:

- Setup and configured MongoDB as a service on Ubuntu
- Accessed MongoDB remotely using [ufw][]{:target="_blank"}

# Adding the Repository
To follow this tutorial you need:

- Ubuntu 16.04 LTS

Ubuntu includes its own MongoDB Community Edition packages, however they are mostly not up-to-date. For this tutorial we will be using
the official MongoDB Community Edition packages which are generally more up-to-date.  
MongoDB only provides packages for `64-bit LTS` (long-term support) Ubuntu releases.

Ubuntu ensures the authenticity of software packages by verifying that they are signed with [GPG][]{:target="_blank"} keys, so we 
first have to import the key for the official MongoDB repository:

{% highlight bash %}
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 0C49F3730359A14518585931BC711F9BA15703C6
{% endhighlight %}

Next we will create a list file for MongoDB. Create the `/etc/apt/sources.list.d/mongodb-org-3.4.list` list file using the following command:

{% highlight bash %}
echo "deb [ arch=amd64,arm64 ] http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.4.list
{% endhighlight %}

We will now reload the local package database:

{% highlight bash %}
sudo apt-get update
{% endhighlight %}

# Installing the MongoDB packages
Now that we got the boring stuff out of the way, let us install MongoDB. Issue the following command to install the latest stable version of MongoDB:

{% highlight bash %}
sudo apt-get install -y mongodb-org
{% endhighlight %}

The MongoDB instance stores its data files in `/var/lib/mongodb` and its log files in `/var/log/mongodb` by default, and runs using the 
`mongodb` user account. You can specify alternate log and data file directories in `/etc/mongod.conf`.

If you change the user that runs the MongoDB process, you **must** modify the access control rights to the `/var/lib/mongodb` and 
`/var/log/mongodb` directories to give this user access to these directories.

Start MongoDB by issuing the following command:

{% highlight bash %}
sudo service mongod start
{% endhighlight %%}

Verify that the [mongod][]{:target="_blank"} process has started successfully by checking the contents of the log file at
`/var/log/mongodb/mongod.log`:

{% highlight bash %}
cat /var/log/mongodb/mongod.log
{% endhighlight %}

Look for a line reading:

{% highlight bash %}
[initandlisten] waiting for connections on port <port>
{% endhighlight %}

where `<port>` is the port configured in `/etc/mongod.conf`, `27017` by default.

## Stopping and Restarting MongoDB
You can stop the [mongod][]{:target="_blank"} process by issuing the following command:

{% highlight bash %}
sudo service mongod stop
{% endhighlight %}

In case MongoDB is running and you want to reload configuration, you restart the [mongod][]{:target="_blank"} process with:

{% highlight bash %}
sudo service mongod restart
{% endhighlight %}

The status of the [mongod][]{:target="_blank"} process can checked with the following command:

{% highlight bash %}
sudo service mongod status
{% endhighlight %}

To enable start of MongoDB when the system starts, run the following command:

{% highlight bash %}
sudo systemctl enable mongod
{% endhighlight %}

# Adjusting the Firewall
In order to access our MongoDB server from a remote system, we need to modify our firewall rules. Ubuntu 16.04 servers can use the
[UFW][]{:target="_blank"} firewall to make sure only connections to certain services are allowed. We can set up a basic firewall
very easily using this application.  
First we list all applications registered with UFW:

{% highlight bash %}
sudo ufw app list
{% endhighlight %}

We need to make sure that the firewall allows SSH connections so that we can log back in next time. We can allow these connections 
by typing:

{% highlight bash %}
sudo ufw allow OpenSSH
{% endhighlight %}

Afterwards, we can enable the firewall by typing:

{% highlight bash %}
sudo ufw enable
{% endhighlight %}

Type `"y"` and press `ENTER` to proceed. You can see that SSH connections are still allowed by typing:

{% highlight bash %}
sudo ufw status
{% endhighlight %}

With this out of the way let us allow port `27017` over the firewall. MongoDB default listening port is `27017`:

{% highlight bash %}
sudo ufw allow 27017
{% endhighlight %}

You can verify the change in firewall settings with `sudo ufw status`.  
You should see traffic to `27017` port allowed in the output. If you have decided to allow only a certain IP address to connect to 
MongoDB server, the IP address of the allowed location will be listed instead of _**Anywhere**_ in the output.

The final bit of configuration is to allow remote connections to MongoDB:

{% highlight bash %}
sudo vi /etc/mongod.conf
{% endhighlight %}

{% highlight bash linenos %}
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# Where and how to store data.
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
#  engine:
#  mmapv1:
#  wiredTiger:

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

# network interfaces
net:
  port: 27017
# bindIp: 127.0.0.1


#processManagement:

#security:

#operationProfiling:

#replication:

#sharding:

## Enterprise-Only Options:

#auditLog:

#snmp:
{% endhighlight %}

On line `24` just below `port: 27017` comment out `bindIp: 127.0.0.1`.  

> **NOTE**  
  It may be a different line number on your system.

Run `sudo service mongod restart` to activate the configuration changes.

# Conclusion
In this blog post, we learned how to setup MongoDB as a service on Ubuntu. We also learned how to enable MongoDB thorugh network firewall.  
I enjoyed writing this post, I hope you did too and until the next post, keep doing cool things :smile:.


[MongoDB]: https://www.mongodb.com/what-is-mongodb
[mongod]: https://docs.mongodb.com/master/reference/program/mongod/#bin.mongod
[Ubuntu]: https://www.ubuntu.com/server
[UFW]: https://help.ubuntu.com/lts/serverguide/firewall.html
[GPG]: https://en.wikipedia.org/wiki/GNU_Privacy_Guard