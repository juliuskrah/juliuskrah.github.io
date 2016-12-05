---
layout:     post
title:      'Building Apache Hadoop from Source on Windows 10 with Visual Studio 2015'
date:       2016-12-05 23:32:12 +0000
categories: blog
tags:       Hadoop Windows
section:    blog
author:     Julius Krah
---
> The [Apache Hadoop][Hadoop]{:target="_blank"}  software library is a framework that allows for the distributed processing of large 
  data sets across clusters of computers using simple programming models. It is designed to scale up from single servers to thousands 
  of machines, each offering local computation and storage. Rather than rely on hardware to deliver high-availability, the library 
  itself is designed to detect and handle failures at the application layer, so delivering a highly-available service on top of a 
  cluster of computers, each of which may be prone to failures.

# Introduction
The easiest way to setup `Hadoop` is to download the binaries from any of the [Apache Download Mirrors][Releases]{:target="_blank"}.
However this only works for GNU/Linux. 

> **NOTE**  
  Hadoop version 2.2 onwards includes native support for Windows.

The official `Apache Hadoop` releases do not include Windows binaries (yet, as at the time of this writing).  
According to the official `Hadoop` [Wiki][]{:target="_blank"}:

> building a Windows package from the sources is fairly straightforward. 

This information however must be taken with a grain of salt as it wasn't _straightforward_ when I built my Windows distribution from 
source. In this post we are going to build `Hadoop` from source on Windows 10 with 
[Visual Studio Community Edition 2015][Visual Studio]{:target="_blank"}.

## Pre-requisites
- Windows 10 64 bit
- [Oracle JDK][JDK]{:target="_blank"} 1.8.0_111+
- [Maven][]{:target="_blank"} 3.3.9+
- [Protocol Buffers][protobuf]{:target="_blank"} 2.5.0
- [CMake][] 3.7.1 +
- [Visual Studio Community Edition 2015][Visual Studio]{:target="_blank"} Update 3+
- [Windows SDK][]{:target="_blank"} 8.1
- [Cygwin][]{:target="_blank"} 2.6.0
- [zlib][]{:target="_blank"} 2.6.0 1.2.8
- Internet connection for first build (to fetch all Maven and Hadoop dependencies)

# Setup Environment
The first step to building `Hadoop` on Windows is to setup our Windows build environment. To set this up, we will define 
the following environment variables:

- `JAVA_HOME`
- `MAVEN_HOME`
- `C:\Protobuf`
- `C:\Program Files\CMake\bin`
- `C:\cygwin64\bin`
- `Platform`
- `ZLIB_HOME`
- `temp`
- `tmp`

> **NOTE**  
  The following assume `JDK 1.8.0_111` is downloaded and installed

The Java installation process on Windows will likely go ahead an install both the JDK and JRE directories (even if the JRE 
wasn't selected during the installation process).  The default installation will install the following directories:  
`C:\Program Files\Java\jdk1.8.0_111`  
`C:\Program Files\Java\jre1.8.0_111`

Some Java programs do not work well with a `JAVA_HOME` environment variable that contains embedded spaces (such as 
`C:\Program Files\java\jdk1.8.0_111`.  To get around this Oracle has created a subdirectory at `C:\ProgramData\Oracle\Java\javapath\` 
to contain links to various Java executables without any embedded spaces, however it has for some reason omitted the `JDK compiler` 
from this list.  To correct for this, we need to create an additional directory symbolic link to the JDK installation.  
Open an Administrative **Command Prompt** by pressing the `Win`{:.fa .fa-windows} key. Type `cmd.exe`, and press `Ctrl + Shift + Enter` 
to open **Command Prompt** in elevated mode. This will provide the proper privileges to create the symbolic link, using the 
following commands:     

{% highlight posh %}
cd C:\ProgramData\Oracle\Java\javapath\    
mklink /D JDK "C:\Program Files\Java\jdk1.8.0_111"
{% endhighlight %}

The `JAVA_HOME` environment System variable can then be set to the  following (with no embedded spaces):  
`JAVA_HOME` ==>  `C:\ProgramData\Oracle\Java\javapath\JDK` 

The Environment Variable editor can be accessed from the "Start" menu by clicking on "Control Panel", then "System and Security", 
then "System", then "Advanced System Settings", then "Environment Variables".  
The `PATH` environment System variable should then be prefixed with the following  
`%JAVA_HOME%\bin` 

Test your installation with:

{% highlight posh %}
javac -version
{% endhighlight %}

Sample output:

{% highlight posh %}
javac 1.8.0_111
{% endhighlight %}

> **NOTE**  
  The following assume `Maven 3.3.9` is downloaded

Extract `Maven` binaries to `C:\apache-maven-3.3.9`. Set the `MAVEN_HOME` environment System variable to the following:
`MAVEN_HOME` ==>  `C:\apache-maven-3.3.9`  
The `PATH` environment System variable should then be prefixed with the following  
`%MAVEN_HOME%\bin`

Test your installation with:

{% highlight posh %}
mvn -v
{% endhighlight %}

Sample output:

{% highlight posh %}
Apache Maven 3.3.9 (bb52c8502b132ec0a5c3f4d09453c07478323dc5; 2015-11-10T16:41:47+00:00)
Maven home: C:\apache-maven-3.3.9\bin\..
Java version: 1.8.0_111, vendor: Oracle Corporation
Java home: C:\ProgramData\Oracle\Java\javapath\JDK\jre
Default locale: en_US, platform encoding: Cp1252
OS name: "windows 10", version: "10.0", arch: "amd64", family: "dos"
{% endhighlight %}

Next extract `Protocol Buffers` to `C:\protobuf`. The `PATH` environment System variable should then be prefixed with the following  
`C:\protobuf`

Test your installation with:

{% highlight posh %}
protoc --version
{% endhighlight %}

Sample output:

{% highlight posh %}
libprotoc 2.5.0
{% endhighlight %}

> **NOTE**  
  The following assume `CMAKE` is installed to `C:\Program Files\CMake`

Next the `PATH` environment System variable should then be prefixed with the following  
`C:\Program Files\CMake\bin`

Test your installation with:

{% highlight posh %}
cmake --version
{% endhighlight %}

Sample output:

{% highlight posh %}
cmake version 3.7.1

CMake suite maintained and supported by Kitware (kitware.com/cmake).
{% endhighlight %}

> **NOTE**
  The following assume `cygwin` is installed to `C:\cygwin64`

Next the `PATH` environment System variable should then be prefixed with the following  
`C:\cygwin64\bin`

Extract the contents of [zlib128-dll][zlib]{:target="_blank"} to `C:\zlib`. This will be needed later.

And that's it for setting up the System Environment variables. Whew! That was a lot of setting up to do.

# Get and Tweak Hadoop Sources
Download the `Hadoop` source files from the [Apache Download Mirrors][Releases]{:target="_blank"}. At the time of this writing, the 
latest `Hadoop` version is [2.7.3][]{:target="_blank"}. Extract the contents of [hadoop-2.7.3-src.tar.gz][2.7.3]{:target="_blank"}
to `C:\hdfs`.

The source files of `Hadoop` are written for `Windows SDK` or `Visual Studio 2010 Professional`. This makes it incompartible with
`Visual Studio 2015 Community Edition`. To work around this, open the following files in `Visual Studio 2015`

- `C:\hdfs\hadoop-common-project\hadoop-common\src\main\winutils\winutils.vcxproj`
- `C:\hdfs\hadoop-common-project\hadoop-common\src\main\winutils\libwinutils.vcxproj`
- `C:\hdfs\hadoop-common-project\hadoop-common\src\main\native\native.vcxproj`

`Visual Studio` will complain of them being of an old version. All you have to do is to save all and close.  
Next enable `cmake Visual Studio` 2015 project generation for `hdfs`. On the line 449 of 
`C:\hdfs\hadoop-hdfs-project\hadoop-hdfs\pom.xml`, edit the `else` value as the following:

{% highlight xml %}
<condition property="generator" value="Visual Studio 10" else="Visual Studio 14 2015 Win64">
{% endhighlight %}

# Build Hadoop
To build `Hadoop` you need the `Developer Command Prompt`. To launch the prompt on Windows 10 follow steps below:

1.    Open the `Start` menu, by pressing the Windows logo *key*{:.fa .fa-windows} on your keyboard for example.
2.    On the `Start` menu, enter **dev**. This will bring a list of installed apps that match your search pattern. 
      If you're looking for a different command prompt, try entering a different search term such as **prompt**.
3.    Choose the `Developer Command Prompt` (or the command prompt you want to use).

From this prompt, add a few more environment variables:

{% highlight posh %}
set Platform=x64
set ZLIB_HOME=C:\zlib\include 
set temp=c:\temp
set tmp=c:\temp
{% endhighlight %}

The `Platform` variable is case sensitive. Finally the build:

{% highlight posh %}
cd \hdfs
mvn package -Pdist,native-win -DskipTests -Dtar
{% endhighlight %}

When everything is successful, we will get an output similar to this:

{% highlight posh %}
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
[INFO]
[INFO] Apache Hadoop Common ............................... SUCCESS [06:51 min]
[INFO] Apache Hadoop NFS .................................. SUCCESS [ 27.131 s]
[INFO] Apache Hadoop KMS .................................. SUCCESS [01:17 min]
[INFO] Apache Hadoop Common Project ....................... SUCCESS [  0.204 s]
[INFO] Apache Hadoop HDFS ................................. SUCCESS [09:33 min]
[INFO] Apache Hadoop HttpFS ............................... SUCCESS [02:52 min]
[INFO] Apache Hadoop HDFS BookKeeper Journal .............. SUCCESS [ 48.980 s]
[INFO] Apache Hadoop HDFS-NFS ............................. SUCCESS [ 18.154 s]
[INFO] Apache Hadoop HDFS Project ......................... SUCCESS [  0.379 s]
[INFO] hadoop-yarn ........................................ SUCCESS [  0.531 s]
[INFO] hadoop-yarn-api .................................... SUCCESS [01:16 min]
[INFO] hadoop-yarn-common ................................. SUCCESS [01:39 min]
[INFO] hadoop-yarn-server ................................. SUCCESS [  0.305 s]
[INFO] hadoop-yarn-server-common .......................... SUCCESS [ 46.543 s]
[INFO] hadoop-yarn-server-nodemanager ..................... SUCCESS [ 51.804 s]
[INFO] hadoop-yarn-server-web-proxy ....................... SUCCESS [ 23.130 s]
[INFO] hadoop-yarn-server-applicationhistoryservice ....... SUCCESS [ 37.183 s]
[INFO] hadoop-yarn-server-resourcemanager ................. SUCCESS [01:07 min]
[INFO] hadoop-yarn-server-tests ........................... SUCCESS [  8.074 s]
[INFO] hadoop-yarn-client ................................. SUCCESS [ 12.261 s]
[INFO] hadoop-yarn-server-sharedcachemanager .............. SUCCESS [ 12.881 s]
[INFO] hadoop-yarn-applications ........................... SUCCESS [  0.070 s]
[INFO] hadoop-yarn-applications-distributedshell .......... SUCCESS [  5.197 s]
[INFO] hadoop-yarn-applications-unmanaged-am-launcher ..... SUCCESS [  4.040 s]
[INFO] hadoop-yarn-site ................................... SUCCESS [  0.186 s]
[INFO] hadoop-yarn-registry ............................... SUCCESS [ 11.515 s]
[INFO] hadoop-yarn-project ................................ SUCCESS [ 17.805 s]
[INFO] hadoop-mapreduce-client ............................ SUCCESS [  0.425 s]
[INFO] hadoop-mapreduce-client-core ....................... SUCCESS [01:20 min]
[INFO] hadoop-mapreduce-client-common ..................... SUCCESS [ 27.595 s]
[INFO] hadoop-mapreduce-client-shuffle .................... SUCCESS [  7.386 s]
[INFO] hadoop-mapreduce-client-app ........................ SUCCESS [ 24.349 s]
[INFO] hadoop-mapreduce-client-hs ......................... SUCCESS [ 16.794 s]
[INFO] hadoop-mapreduce-client-jobclient .................. SUCCESS [ 38.134 s]
[INFO] hadoop-mapreduce-client-hs-plugins ................. SUCCESS [  3.672 s]
[INFO] Apache Hadoop MapReduce Examples ................... SUCCESS [ 21.488 s]
[INFO] hadoop-mapreduce ................................... SUCCESS [ 13.625 s]
[INFO] Apache Hadoop MapReduce Streaming .................. SUCCESS [ 27.357 s]
[INFO] Apache Hadoop Distributed Copy ..................... SUCCESS [01:31 min]
[INFO] Apache Hadoop Archives ............................. SUCCESS [  4.110 s]
[INFO] Apache Hadoop Rumen ................................ SUCCESS [ 12.072 s]
[INFO] Apache Hadoop Gridmix .............................. SUCCESS [  9.479 s]
[INFO] Apache Hadoop Data Join ............................ SUCCESS [  4.361 s]
[INFO] Apache Hadoop Ant Tasks ............................ SUCCESS [  3.888 s]
[INFO] Apache Hadoop Extras ............................... SUCCESS [  5.252 s]
[INFO] Apache Hadoop Pipes ................................ SUCCESS [  0.062 s]
[INFO] Apache Hadoop OpenStack support .................... SUCCESS [  6.984 s]
[INFO] Apache Hadoop Amazon Web Services support .......... SUCCESS [01:13 min]
[INFO] Apache Hadoop Azure support ........................ SUCCESS [ 14.862 s]
[INFO] Apache Hadoop Client ............................... SUCCESS [ 10.688 s]
[INFO] Apache Hadoop Mini-Cluster ......................... SUCCESS [  1.479 s]
[INFO] Apache Hadoop Scheduler Load Simulator ............. SUCCESS [  9.714 s]
[INFO] Apache Hadoop Tools Dist ........................... SUCCESS [ 10.291 s]
[INFO] Apache Hadoop Tools ................................ SUCCESS [  0.058 s]
[INFO] Apache Hadoop Distribution ......................... SUCCESS [01:03 min]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 39:51 min
[INFO] Finished at: 2016-12-04T16:47:41+00:00
[INFO] Final Memory: 208M/809M
[INFO] ------------------------------------------------------------------------
{% endhighlight %}

This will build the binaries to `C:\hdfs\hadoop-dist\target\hadoop-2.7.3.tar.gz`.

# Install Hadoop
With our build successful, we can now install `Hadoop` on Windows 10. Pick a target directory for installing the package. 
We use `C:\hadoop` as an example. Extract the `tar.gz` file (e.g.hadoop-2.7.3.tar.gz) under `C:\hadoop`. This will yield a 
directory structure like the following. If installing a multi-node cluster (We will cover this in a different post), 
then repeat this step on every node:

{% highlight posh %}
C:\hadoop>dir
 Volume in drive C is Windows
 Volume Serial Number is 0627-71D6

 Directory of C:\hadoop

12/04/2016  22:49    <DIR>          .
12/04/2016  22:49    <DIR>          ..
12/04/2016  22:14    <DIR>          bin
12/04/2016  22:14    <DIR>          etc
12/04/2016  22:14    <DIR>          include
12/04/2016  22:14    <DIR>          libexec
12/04/2016  16:46            84,854 LICENSE.txt
12/04/2016  22:41    <DIR>          logs
12/04/2016  16:46            14,978 NOTICE.txt
12/04/2016  16:46             1,366 README.txt
12/04/2016  22:14    <DIR>          sbin
12/04/2016  22:14    <DIR>          share
               3 File(s)        101,198 bytes
               9 Dir(s)  231,546,564,608 bytes free
{% endhighlight %} 

This section describes the absolute minimum configuration required to start a Single Node (pseudo-distributed) cluster.

Add Environment Variable `HADOOP_HOME` and edit `Path` Variable to add `bin` directory of `HADOOP_HOME` (say `%HADOOP_HOME%\bin`).  
Before you can start the `Hadoop` Daemons you will need to make a few edits to configuration files. The configuration file 
templates will all be found in `C:\hadoop\etc\hadoop`, assuming your installation directory is `C:\hadoop`. 

> Follow the following to edit the HDFS Configuration

First edit the file **hadoop-env.cmd** to add the following lines near the end of the file:

{% highlight posh %}
set HADOOP_PREFIX=C:\hadoop
set HADOOP_CONF_DIR=%HADOOP_PREFIX%\etc\hadoop
set YARN_CONF_DIR=%HADOOP_CONF_DIR%
set PATH=%PATH%;%HADOOP_PREFIX%\bin
{% endhighlight %} 

Edit the file **core-site.xml** and make sure it has the following configuration key: 

{% highlight xml %}
<configuration>
  <property>
    <name>fs.default.name</name>
    <value>hdfs://0.0.0.0:19000</value>
  </property>
</configuration>
{% endhighlight %}

> **fs.default.name**:  
  The name of the default file system. A URI whose scheme and authority determine the FileSystem implementation. 
  The uri's scheme determines the config property (fs.SCHEME.impl) naming the FileSystem implementation class. 
  The uri's authority is used to determine the host, port, etc. for a filesystem.

Edit the file **hdfs-site.xml** and add the following configuration key:

{% highlight xml %}
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
</configuration>
{% endhighlight %}

> **dfs.replication**:  
  Default block replication. The actual number of replications can be specified when the file is created. The default is used if 
  replication is not specified in create time.

Finally, edit the file **slaves** and make sure it has the following entry:

{% highlight posh %}
localhost
{% endhighlight %}

The default configuration puts the HDFS metadata and data files under `\tmp` on the current drive. In the above example 
this would be `C:\tmp`. For your first test setup you can just leave it at the default.

> Follow the following to edit the YARN Configuration

Edit **mapred-site.xml** under `%HADOOP_PREFIX%\etc\hadoop`:

{% highlight posh %}
copy mapred-site.xml.template mapred-site.xml
{% endhighlight %}

Add the following entries:

{% highlight xml %}
<configuration>
   <property>
     <name>mapreduce.framework.name</name>
     <value>yarn</value>
   </property>
</configuration> 
{% endhighlight %}

> **mapreduce.framework.name**:  
  The runtime framework for executing MapReduce jobs. Can be one of local, classic or yarn.

Finally, edit **yarn-site.xml** and add the following entries:

{% highlight xml %} 
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>

  <property>
    <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
  </property>

  <property>
    <name>yarn.application.classpath</name>
    <value>%HADOOP_CONF_DIR%,%HADOOP_COMMON_HOME%/share/hadoop/common/*,%HADOOP_COMMON_HOME%/share/hadoop/common/lib/*,%HADOOP_HDFS_HOME%/share/hadoop/hdfs/*,%HADOOP_HDFS_HOME%/share/hadoop/hdfs/lib/*,%HADOOP_MAPRED_HOME%/share/hadoop/mapreduce/*,%HADOOP_MAPRED_HOME%/share/hadoop/mapreduce/lib/*,%HADOOP_YARN_HOME%/share/hadoop/yarn/*,%HADOOP_YARN_HOME%/share/hadoop/yarn/lib/*</value>
  </property>
</configuration>
{% endhighlight %}

> **yarn.nodemanager.aux-services**:  
  The auxiliary service name. Default value is omapreduce_shuffle  
  **yarn.nodemanager.aux-services.mapreduce.shuffle.class**:  
  The auxiliary service class to use. Default value is org.apache.hadoop.mapred.ShuffleHandler  
  **yarn.application.classpath**:  
  CLASSPATH for YARN applications. A comma-separated list of CLASSPATH entries.

Run 

{% highlight posh %}
C:\deploy\etc\hadoop\hadoop-env.cmd
{% endhighlight %}

to setup environment variables that will be used by the startup scripts and the daemons. 

Format the filesystem with the following command:

{% highlight posh %}
hdfs namenode -format
{% endhighlight %}

This command will print a number of filesystem parameters. Just look for the following two strings to ensure that the 
format command succeeded:

{% highlight posh %}
16/12/05 22:05:35 INFO namenode.FSImageFormatProtobuf: Saving image file \tmp\hadoop-username\dfs\name\current\fsimage.ckpt_0000000000000000000 using no compression
16/12/05 22:05:35 INFO namenode.FSImageFormatProtobuf: Image file \tmp\hadoop-username\dfs\name\current\fsimage.ckpt_0000000000000000000 of size 355 bytes saved in 0 seconds.
{% endhighlight %}

Run the following command to start the NameNode and DataNode on localhost:

{% highlight posh %}
%HADOOP_PREFIX%\sbin\start-dfs.cmd
{% endhighlight %}

Two separate Command Prompt windows will be opened automatically to run **Namenode** and **Datanode**.

To verify that the HDFS daemons are running, we would create a file:

{% highlight posh %}
echo > myfile.txt
{% endhighlight %}

Try copying the file to HDFS:

{% highlight posh %}
%HADOOP_PREFIX%\bin\hdfs dfs -put myfile.txt /
%HADOOP_PREFIX%\bin\hdfs dfs -ls /
{% endhighlight %}

Sample output:

{% highlight posh %}
Found 1 items
drwxr-xr-x   - username supergroup          4640 2014-01-18 08:40 /myfile.txt
{% endhighlight %}

Finally, start the YARN daemons:

{% highlight posh %}
%HADOOP_PREFIX%\sbin\start-yarn.cmd
{% endhighlight %}

Similarly, two separate Command Prompt windows will be opened automatically to run **Resource Manager** and **Node Manager**.
The cluster should be up and running! To verify, we can run a simple wordcount job on the text file we just copied to HDFS:

{% highlight posh %}
%HADOOP_PREFIX%\bin\yarn jar %HADOOP_PREFIX%\share\hadoop\mapreduce\hadoop-mapreduce-example
s-2.5.0.jar wordcount /myfile.txt /out
{% endhighlight %}

If everything goes well then you will be able to open the **Resource Manager** and **Node Manager** at 
[http://localhost:8042](http://localhost:8042){:target="_blank"} and **Namenode** at [http://localhost:50070](http://localhost:50070){:target="_blank"}.

Stop HDFS & MapReduce with the following commands:

{% highlight posh %}
%HADOOP_PREFIX%\sbin\stop-dfs.cmd
%HADOOP_PREFIX%\sbin\stop-yarn.cmd
{% endhighlight %}

# Conclusion
In this post, we built `Hadoop` from source on Windows 10 64 bit with `Visual Studio 2015 Community Edition` albeit a tideous process. 
We also setup a single `Hadoop` cluster on Windows. Until the next post, keep doing cool things :smile:.


[Hadoop]: http://hadoop.apache.org/
[Releases]: http://www.apache.org/dyn/closer.cgi/hadoop/common/
[Wiki]: https://wiki.apache.org/hadoop/Hadoop2OnWindows
[JDK]: http://www.oracle.com/technetwork/java/javase/downloads/index-jsp-138363.html
[Maven]: http://maven.apache.org/download.cgi
[Protobuf]: https://github.com/google/protobuf/releases/download/v2.5.0/protoc-2.5.0-win32.zip
[Visual Studio]: https://go.microsoft.com/fwlink/?LinkId=615448&clcid=0x409
[CMake]: https://cmake.org/files/v3.7/cmake-3.7.1-win64-x64.msi
[Windows SDK]: https://developer.microsoft.com/en-us/windows/downloads/windows-8-1-sdk
[Cygwin]: https://cygwin.com/setup-x86_64.exe
[zlib]: http://prdownloads.sourceforge.net/libpng/zlib128-dll.zip?download
[2.7.3]: http://www-eu.apache.org/dist/hadoop/common/hadoop-2.7.3/hadoop-2.7.3-src.tar.gz