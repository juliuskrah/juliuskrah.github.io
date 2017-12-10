---
layout:     post
title:      Developing RESTful Services with Flask
date:       2017-12-10 09:37:07 +0000
categories: blog
tags:       python pip rest flask
section:    blog
author:     juliuskrah
repo:       /rest-service-flask/tree/master
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
Create files and directories as outlined below before you continue. If you are uncertain view the source
{% include source.html %}.

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

In the next section we will discuss how to setup a REST service in Flask using a Flask extension library:
[`Flask RESTful`][Flask RESTful]{:target="_blank"}

# Setting Up
Before we set up the project we need to create a virtual environment for Python. 
[`virtualenv`][virtualenv]{:target="_blank"} is a tool to create isolated Python environments.
Install `virtualenv` with [`pip`][pip]{:target="_blank"}:

{% highlight posh %}
> pip install virtualenv
{% endhighlight %}

We can then create an isolated Python environment with:

{% highlight posh %}
> virtualenv venv
{% endhighlight %}

Where `venv` is a directory to place the new virtual environment.

In a newly created `virtualenv` there will also be a activate shell script. For Windows systems, activation 
scripts are provided for the Command Prompt and Powershell.
On Posix systems, this resides in `/ENV/bin/`, so you can run:

{% highlight bash %}
$ source bin/activate
$ activate
{% endhighlight %}

To undo these changes, just run:

{% highlight bash %}
$ deactivate
{% endhighlight %}

On Windows, the equivalent activate script is in the Scripts folder:

{% highlight posh %}
> \path\to\venv\Scripts\activate
{% endhighlight %}

And type `deactivate` to undo the changes. Now _**activate**_ the virtual environement using the methods 
outlined above based on your operating system.

We will organise the dependencies for the REST application in a `requirements.txt` file:

file: {% include file-path.html file_path='requirements.txt' %}

```
flask-restful
python-simplexml
```

If it is not obvious at this point, `flask-restful` is flask extension for RESTful webservices and `SimpleXML` 
is a library for very simply manipulating XML files. It lets you harvest data from XML files, change values of attributes, print or change data of elements, create new elements, etc.

Install the packages with the following command:

{% highlight posh %}
> pip install -r requirements.txt
{% endhighlight %}

This pulls in `Flask RESTful` and `SimpleXML` and all their dependent packages.

# Creating Utility Module for REST
We will create a small utility module to assist with `GET`, `UPDATE`, `POST` and `DELETE` operations:

file: {% include file-path.html file_path='todoapi/common/util.py' %}

{% highlight python %}
from flask_restful import abort
from datetime import datetime

TODOS = [
    {'id': 1, 'description': 'Create a post on REST using Flask',
        'created_time': str(datetime.now()), 'modified_time': None},
    {'id': 2, 'description': 'Store REST data in a database for Flask post',
        'created_time': str(datetime.now()), 'modified_time': None},
    ...
    {'id': 8, 'description': 'Write a flask tutorial for WebSockets',
        'created_time': str(datetime.now()), 'modified_time': None},
    {'id': 9, 'description': 'Form handling in Flask',
        'created_time': str(datetime.now()), 'modified_time': None},
    {'id': 10, 'description': 'Handling binary files in flask',
        'created_time': str(datetime.now()), 'modified_time': None},
]

def remove_todo(key):
    todo = find_todo(TODOS, key)
    TODOS.remove(todo)

def update_todo(todo, key):
    for idx, item in enumerate(TODOS):
        if key is item['id']:
            todo['id'] = key
            todo['modified_time'] = str(datetime.now())
            todo['created_time'] = item['created_time']
            TODOS[idx] = todo

def add_todo(todo):
    todo['created_time'] = str(datetime.now())
    TODOS.append(todo)

def find_todo(seq, key):
    return next((item for item in seq if item["id"] == key), None)

def abort_if_todo_doesnt_exist(todo_id):
    if find_todo(TODOS, todo_id) is None:
        abort(404, message="Todo {} doesn't exist".format(todo_id))
{% endhighlight %}

In the above snippet we added `abort_if_todo_doesnt_exist` function to handle 404 errors when a resource with
the given id does not exist in the `TODOS` array.


# Create the Resource Classes
Next we create the resource classes that consume the utility module:

file: {% include file-path.html file_path='todoapi/resources/todo_resource.py' %}

{% highlight python %}
from flask_restful import Resource
from flask import request
from todoapi.common import util

class Todo(Resource):
    def get(self, todo_id):
        util.abort_if_todo_doesnt_exist(todo_id)
        return util.find_todo(util.TODOS, todo_id)

    def delete(self, todo_id):
        util.abort_if_todo_doesnt_exist(todo_id)
        util.remove_todo(todo_id)
        return '', 204

    def put(self, todo_id):
        util.abort_if_todo_doesnt_exist(todo_id)
        task = request.get_json(force=True)
        util.update_todo(task, todo_id)
        return task, 201

class TodoList(Resource):
    def get(self):
        return util.TODOS

    def post(self):
        task = request.get_json(force=True)
        util.add_todo(task)
        return task, 201
{% endhighlight %}

In this module we have two resource classes. `Todo` serves single resources, and `TodoList` serves an array.


# Wrapping up
It is time to create our entry point module to start our flask REST service:

file: {% include file-path.html file_path='todoapi/app.py' %}

{% highlight python linenos %}
rom simplexml import dumps
from flask import make_response, Flask
from flask_restful import Api
from todoapi.resources import todo_resource

def output_xml(data, code, headers=None):
    """Makes a Flask response with a XML encoded body"""
    resp = make_response(dumps({'response': data}), code)
    resp.headers.extend(headers or {})
    return resp

app = Flask(__name__)
api = Api(app, default_mediatype='application/json')
api.representations['application/xml'] = output_xml

api.add_resource(todo_resource.TodoList, '/api/v1.0/resources')
api.add_resource(todo_resource.Todo, '/api/v1.0/resources/<int:todo_id>')
{% endhighlight %}

`output_xml` is a marshaller for XML. On `line 14` we are telling flask to use this marshaller for XML
representation. On `lines 16 and 17` we map the endpoints to the representations.

## Running and Testing the Endpoints
To run the service you need to set two environment variables:

{% highlight posh %}
> set FLASK_APP=app.py    # export FLASK_APP=app.py -- on Unix
> set FLASK_DEBUG=1       # export FLASK_DEBUG=1    -- on Unix
{% endhighlight %}

Then activate the `virtualenv` if you haven't already done so. Navigate to the `todoapi` directory i.e.
`cd todoapi` and run the following command:

{% highlight posh %}
> flask run
{% endhighlight %}

Using `cURL` or a REST client of your choice test the `GET` endpoint:

{% highlight posh %}
> curl -i -X GET http://127.0.0.1:5000/api/v1.0/resources/

HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 1323
Server: Werkzeug/0.12.2 Python/3.5.2
Date: Sun, 10 Dec 2017 08:28:04 GMT

[{
    "modified_time": null,
    "created_time": "2017-12-10 08:27:06.557315",
    "description": "Create a post on REST using Flask",
    "id": 1
}, {
    "modified_time": null,
    "created_time": "2017-12-10 08:27:06.557315",
    "description": "Store REST data in a database for Flask post",
    "id": 2
}, {
    "modified_time": null,
    "created_time": "2017-12-10 08:27:06.557315",
    "description": "Secure a REST service in Flask",
    "id": 3
}, {
    "modified_time": null,
    "created_time": "2017-12-10 08:27:06.557315",
    "description": "Write another article on Flask",
    "id": 4
}, {
    "modified_time": null,
    "created_time": "2017-12-10 08:27:06.557315",
    "description": "Secure a flask REST service using JWT",
    "id": 5
}, {
    "modified_time": null,
    "created_time": "2017-12-10 08:27:06.557315",
    "description": "Secure a REST service in Flask using OAuth2",
    "id": 6
}, {
    "modified_time": null,
    "created_time": "2017-12-10 08:27:06.557315",
    "description": "SQL Migration using Flask",
    "id": 7
}, {
    "modified_time": null,
    "created_time": "2017-12-10 08:27:06.557315",
    "description": "Write a flask tutorial for WebSockets",
    "id": 8
}, {
    "modified_time": null,
    "created_time": "2017-12-10 08:27:06.557315",
    "description": "Form handling in Flask",
    "id": 9
}, {
    "modified_time": null,
    "created_time": "2017-12-10 08:27:06.557315",
    "description": "Handling binary files in flask",
    "id": 10
}]
{% endhighlight %}

You can also view the representation as XML:

{% highlight posh %}
> curl -i -H "Accept: application/xml" -X GET http://127.0.0.1:5000/api/v1.0/resources/

HTTP/1.0 200 OK
Content-Type: application/xml
Content-Length: 1645
Server: Werkzeug/0.12.2 Python/3.5.2
Date: Sun, 10 Dec 2017 08:33:00 GMT

<?xml version="1.0" ?>
<response>
    <modified_time>None</modified_time>
    <created_time>2017-12-10 08:27:06.557315</created_time>
    <description>Create a post on REST using Flask</description>
    <id>1</id>
    <modified_time>None</modified_time>
    <created_time>2017-12-10 08:27:06.557315</created_time>
    <description>Store REST data in a database for Flask post</description>
    <id>2</id>
    <modified_time>None</modified_time>
    <created_time>2017-12-10 08:27:06.557315</created_time>
    <description>Secure a REST service in Flask</description>
    <id>3</id>
    <modified_time>None</modified_time>
    <created_time>2017-12-10 08:27:06.557315</created_time>
    <description>Write another article on Flask</description>
    <id>4</id>
    <modified_time>None</modified_time>
    <created_time>2017-12-10 08:27:06.557315</created_time>
    <description>Secure a flask REST service using JWT</description>
    <id>5</id>
    <modified_time>None</modified_time>
    <created_time>2017-12-10 08:27:06.557315</created_time>
    <description>Secure a REST service in Flask using OAuth2</description>
    <id>6</id>
    <modified_time>None</modified_time>
    <created_time>2017-12-10 08:27:06.557315</created_time>
    <description>SQL Migration using Flask</description>
    <id>7</id>
    <modified_time>None</modified_time>
    <created_time>2017-12-10 08:27:06.557315</created_time>
    <description>Write a flask tutorial for WebSockets</description>
    <id>8</id>
    <modified_time>None</modified_time>
    <created_time>2017-12-10 08:27:06.557315</created_time>
    <description>Form handling in Flask</description>
    <id>9</id>
    <modified_time>None</modified_time>
    <created_time>2017-12-10 08:27:06.557315</created_time>
    <description>Handling binary files in flask</description>
    <id>10</id>
</response>
{% endhighlight %}

We can get a single item using `curl -i -X GET http://127.0.0.1:5000/api/v1.0/resources/1`.
To delete use `curl -i -X DELETE http://127.0.0.1:5000/api/v1.0/resources/1`.

The `POST` and `PUT` endpoints require a body:

{% highlight posh%}
> curl -i -X POST http://127.0.0.1:5000/api/v1.0/resources/ -d "{\"id\": 11,  \"description\": \"New Todo\"}"

HTTP/1.0 201 CREATED
Content-Type: application/json
Content-Length: 84
Server: Werkzeug/0.12.2 Python/3.5.2
Date: Sun, 10 Dec 2017 09:03:26 GMT

{
    "created_time": "2017-12-10 09:03:26.744768",
    "description": "New Todo",
    "id": 11
}
{% endhighlight %}

{% highlight posh%}
> curl -i -X PUT http://127.0.0.1:5000/api/v1.0/resources/2 -d "{\"description\": \"Update todo item 2\"}"

HTTP/1.0 201 CREATED
Content-Type: application/json
Content-Length: 140
Server: Werkzeug/0.12.2 Python/3.5.2
Date: Sun, 10 Dec 2017 09:06:25 GMT

{
    "created_time": "2017-12-10 08:57:40.635545",
    "description": "Update todo item 2",
    "id": 2,
    "modified_time": "2017-12-10 09:06:25.950571"
}
{% endhighlight %}

# Conclusion
In this post we learned how to create a REST service with Flask in Python.     
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.

[Flask]:                            http://flask.pocoo.org/
[Flask RESTful]:                    http://flask-restful.readthedocs.io/en/latest/
[pip]:                              https://pypi.python.org/pypi/pip/
[Python]:                           https://www.python.org/
[virtualenv]:                       https://pypi.python.org/pypi/virtualenv