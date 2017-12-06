---
layout:     post
title:      Developing RESTful Services with Flask
categories: blog
tags:       python pip rest flask
section:    blog
author:     juliuskrah
repo:       rest-example/tree/flask
---
> [Flask][]{:target="_blank"} is a microframework for Python based on Werkzeug, Jinja 2 and good intentions.

# Introduction
In this blog post I will introduce you to the `Flask` Framework and how to build a simple `To Do` REST service.

"Micro" does not mean that your whole web application has to fit into a single Python file (although it 
certainly can), nor does it mean that Flask is lacking in functionality. The "micro" in microframework means 
`Flask` aims to keep the core simple but extensible. Flask won't make many decisions for you, such as what 
database to use. Those decisions that it does make, such as what templating engine to use, are easy to change.
Everything else is up to you, so that Flask can be everything you need and nothing you don't.

A minimal Flask application looks something like this:

{% highlight python %}
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'
{% endhighlight %}

## Directory Structure
```
.
|__todoapi/
|  |__`__init__.py
|  |__app.py
|  |__resources/
|  |  |__`__init__.py
|  |  |__todo_resource.py
|  |__common/
|  |  |__`init__.py
|  |  |__util.py
|__venv/
|__requirements.txt
```

## Prerequisites
- [Python][]{:target="_blank"}
- [pip][]{:target="_blank"}
- [Flask][]{:target="_blank"}

In the next section we will discuss how to setup a REST service in Flask using a Flask extension library:
[`Flask RESTful`][Flask RESTful]{:target="_blank"}

# Setting Up
Before we set up the project we need to create a virtual environment for Python. 
[`virtualenv`][virtualenv]{:target="_blank"} is a tool to create isolated Python environments.
Install `virtualenv` with [`pip`][pip]{:target="_blank"}:

{% highlight bash %}
> pip install virtualenv
{% endhighlight %}

We can then create an isolated Python environment with:

{% highlight bash %}
> virtualenv venv
{% endhighlight %}

Where `venv` is a directory to place the new virtual environment.

In a newly created `virtualenv` there will also be a activate shell script. For Windows systems, activation 
scripts are provided for the Command Prompt and Powershell.
On Posix systems, this resides in /ENV/bin/, so you can run:

{% highlight bash %}
> source bin/activate
{% endhighlight %}

To undo these changes, just run:

{% highlight bash %}
> deactivate
{% endhighlight %}

On Windows, the equivalent activate script is in the Scripts folder:

{% highlight posh %}
> \path\to\venv\Scripts\activate
{% endhighlight %}

And type `deactivate` to undo the changes. Now activate the virtual environement using the methods outlined 
above based on your operating environment.

We will organise the dependencies for the REST application in a `requirements.txt` file:

file: {% include file-path.html file_path='requirements.txt' %}

```
flask-restful
```

Install the packages with the following command:

{% highlight posh %}
> pip install -r requirements.txt
{% endhighlight %}

This pulls in `Flask RESTful` and all its dependent packages.

# Next Section



[Flask]:                            http://flask.pocoo.org/
[Flask RESTful]:                    http://flask-restful.readthedocs.io/en/latest/
[pip]:                              https://pypi.python.org/pypi/pip/
[Python]:                           https://www.python.org/
[virtualenv]:                       https://pypi.python.org/pypi/virtualenv