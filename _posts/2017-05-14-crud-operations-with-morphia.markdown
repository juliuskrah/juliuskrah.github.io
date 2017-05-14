---
layout:     post
title:      'CRUD operations with Morphia'
date:       2017-05-14 00:06:48 +0000
categories: blog
tags:       java gradle mongodb morphia spring-boot
section:    blog
author:     juliuskrah
repo:       morphia-example/tree/morphia-crud
---
> [Morphia][]{:target="_blank"} is the Java Object Document Mapper for [MongoDB][]{:target="_blank"}

# Introduction
In this post we are going to examine `Morphia` as an Oject Document Mapper (ODM) tool for `MongoDB`. We will map Java
Objects to MongoDB Documents. We will also perform basic Create, Read, Update and Delete (CRUD) with `Morphia`.  
To follow along this post, you should have a working installation of MongoDB on [Windows]({% post_url 2016-11-26-setting-up-mongodb-on-windows %})
and / or [Linux]({% post_url 2017-01-26-setting-up-mongodb-on-ubuntu-16-04 %}).

## Prerequisites
- [Java Development Kit][JDK]{:target="_blank"}
- [Gradle][]{:target="_blank"}
- [MongoDB][]{:target="_blank"}

## Project Structure
This is a `gradle` based project and we are using the standard Java project structure.  

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |juliuskrah/
|  |  |  |  |__morphia/    
|  |  |  |  |  |__Application.java
|  |  |  |  |  |__config/
|  |  |  |  |  |  |__DataSourceConfig.java
|  |  |  |  |  |__entity/
|  |  |  |  |  |  |__Author.java
|  |  |  |  |  |  |__Book.java
|  |  |  |  |  |__repository/
|  |  |  |  |  |  |__AuthorRepository.java
|  |  |  |  |  |  |__BookRepository.java
|  |  |  |  |  |  |__impl/
|  |  |  |  |  |  |  |__AuthorRepositoryImpl.java
|  |  |  |  |  |  |  |__BookRepositoryImpl.java
|  |  |__resources/
|  |  |  |__application.yaml
|__build.gradle
```

## Project Dependencies
Before we can use `Morphia` we need to add the dependency in our gradle buildscript.

file: `build.gradle`

{% highlight gradle %}
...
dependencies {
  compile 'org.mongodb.morphia:morphia:1.3.2'
}
{% endhighlight %}

To illustrate the use of `Morphia` we will build a `Spring-Boot` application.

file: `build.gradle`

{% highlight gradle %}
...
dependencies {
  compile 'org.springframework.boot:spring-boot-starter:1.5.3.RELEASE'
}
{% endhighlight %}

Our full `build.gradle` file:

{% highlight gradle %}
buildscript {
  ext {
    springBootVersion = '1.5.3.RELEASE'
  }
  repositories {
    mavenCentral()
  }
  dependencies {
    classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
  }
}

apply plugin: 'java'
apply plugin: 'org.springframework.boot'

jar {
  baseName = 'morphia-sample'
  version = '0.0.2-SNAPSHOT'
}
sourceCompatibility = 1.8
targetCompatibility = 1.8

repositories {
  mavenCentral()
}

dependencies {
  compile 'org.springframework.boot:spring-boot-starter'
  compile 'org.mongodb.morphia:morphia:1.3.2'
}
{% endhighlight %}

# Configure Spring Application
Let us start with setting up our Spring-Boot application

file: `src/main/java/com/juliuskrah/morphia/Application.java`

{% highlight java %}
@SpringBootApplication
public class Application {

  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }
}
{% endhighlight %}

The above snippet starts up Spring-Boot with auto-configuration enabled.  
Next we configure `Morphia` for Spring-Boot.

file: `src/main/java/com/juliuskrah/morphia/config/DataSourceConfig.java`

{% highlight java %}
@Configuration
public class DataSourceConfig {
  // Inject an instance of Spring-Boot MongoProperties
  @Autowired
  private MongoProperties mongoProperties;

  private Morphia morphia() {
    final Morphia morphia = new Morphia();
    // tell Morphia where to find your classes
    // can be called multiple times with different packages or classes
    morphia.mapPackage("com.juliuskrah.morphia.entity");

    return morphia;
  }

  @Bean
  public Datastore datastore(MongoClient mongoClient) {
    // create the Datastore connecting to the default port on the local host
    final Datastore datastore = morphia().createDatastore(mongoClient, mongoProperties.getDatabase());
    datastore.ensureIndexes();

    return datastore;
  }
}
{% endhighlight %}

Finally add the externalized configuration for the locally installed MongoDB database.

file: `src\main\resources\application.yaml`

{% highlight yaml %}
spring:
  data:
    mongodb:
      database: morphia

# Optional [Add these only if your MongoDB instance is secured]
      authentication-database: some-database
      username: some-user
      password: secret
{% endhighlight %}

# Object Document Mapping
With all the basic requirements out of the way, we can start mapping our Java Objects to MongoDB documents using
an `Author` to `Book` example. An author can write many books, and a book can have multiple authors. We will keep
this simple by focusing on an author with many books and not the inverse relationship.

There are two ways that Morphia can handle your classes: as top level entities or embedded in others. Any class
annotated with `@Entity` is treated as a top level document stored directly in a collection. Any class with `@Entity`
must have a field annotated with `@Id` to define which field to use as the `_id` value in the document written to
MongoDB. `@Embedded` indicates that the class will result in a subdocument inside another document. `@Embedded` classes
do not require the presence of an `@Id` field. We would not cover `@Embedded` in this post.

file: `src/main/java/com/juliuskrah/morphia/entity/Book.java`

{% highlight java %}
@Entity("books")
public class Book {
  @Id
  private ObjectId id;
  private String title;
  @Property("published")
  private LocalDate publicationDate;

  // No args Constructor

  public Book(String title, LocalDate publicationDate) {
    this.title = title;
    this.publicationDate = publicationDate;
  }

  // Getters and Setters omitted for brevity
}
{% endhighlight %}

file: `src/main/java/com/juliuskrah/morphia/entity/Author.java`

{% highlight java %}
@Entity("authors")
@Indexes({
    @Index(fields = { @Field("name") })
  }
)
public class Author {
  @Id
  private ObjectId id;
  private String name;
  @Reference
  private Set<Book> books;

  // No args Constructor
  public Author(String name) {
    this.name = name;
  }

  // Getters and Setters omitted for brevity
}
{% endhighlight %}

There are a few things here to discuss and others we will defer to later sections. The above classes are annotated using
the `@Entity` annotation so we know that it will be a top level document. In the annotation, you’ll see `"books"` and
`"authors"`. By default, Morphia will use the class name as the collection name. If you pass a String instead, it will
use that value for the collection name. In this case, all `Book` instances will be saved in to the `books` collection
and all `Author` instances to the `authors` collection instead. 

The `@Indexes` annotation lists which indexes Morphia should create. In this instance, we’re defining an index named 
`name` on the field `name` with the default ordering of ascending.

We have marked the `id` field to be used as our primary key (the `_id` field in the document). In this instance we are
using the Java driver type of `ObjectId` as the ID type. The ID can be any type you would like but is generally
something like `ObjectId` or `Long`. There are two other annotations to cover but it should be pointed out now that
other than transient and static fields, Morphia will attempt to copy every field to a document bound for the database.

The simplest of the two remaining annotations is `@Property`. This annotation is entirely optional. If you leave this
annotation off, Morphia will use the Java field name as the document field name. Often times this is fine. However,
some times you will want to change the document field name for any number of reasons. In those cases, you can use 
`@Property` and pass it the name to be used when this class is serialized out to a document to be handed off to MongoDB.

This just leaves `@Reference`. This annotation is telling Morphia that this field refers to other Morphia mapped 
entities. In this case Morphia will store what MongoDB calls a [`DBRef`][DBRef]{:target="_blank"} which is just a 
collection name and key value. These referenced entities must already be saved or at least have an ID assigned or
Morphia will throw an exception.

## Creating the CRUD repository
We will create a simple repository class to take care of our CRUD operations.

file: `src/main/java/com/juliuskrah/morphia/repository/impl/AuthorRepositoryImpl.java`

{% highlight java %}
@Repository
public class AuthorRepositoryImpl {

  @Autowired
  private Datastore datastore;

  public Key<Author> create(Author author) {
    return datastore.save(author);
  }

  public Author read(ObjectId id) {
    return datastore.get(Author.class, id);
  }

  public UpdateResults update(Author author, UpdateOperations<Author> operations) {
    return datastore.update(author, operations);
  }

  public WriteResult delete(Author author) {
    return datastore.delete(author);
  }

  public UpdateOperations<Author> createOperations() {
    return datastore.createUpdateOperations(Author.class);
  }
}
{% endhighlight %}

file: `src/main/java/com/juliuskrah/morphia/repository/impl/BookRepositoryImpl.java`

{% highlight java %}
@Repository
public class BookRepositoryImpl {
  @Autowired
  private Datastore datastore;

  public Key<Book> create(Book book) {
    return datastore.save(book);
  }

  public Book read(ObjectId id) {
    return datastore.get(Book.class, id);
  }

  public UpdateResults update(Book book, UpdateOperations<Book> operations) {
    return datastore.update(book, operations);
  }

  public WriteResult delete(Book book) {
    return datastore.delete(book);
  }

  public UpdateOperations<Book> createOperations() {
    return datastore.createUpdateOperations(Book.class);
  }
}
{% endhighlight %}

# Creating Documents
For the most part, you treat your Java objects just like you normally would. When you’re ready to write an object to 
the database, it’s as simple as this:

{% highlight java %}
BookRepository bookRepo =     // get instance;
AuthorRepository authorRepo = // get instance;
		
Book ci = new Book("Continous Integration", LocalDate.now());
// id will be generated after save
bookRepo.create(ci);
		
Author julius = new Author("Julius");
// referenced entities must already be saved or at least have an ID assigned or Morphia will throw an exception
julius.setBooks(Stream.of(ci).collect(Collectors.toSet()));
authorRepo.create(julius);	
{% endhighlight %}

# Reading Documents
Morphia attempts to make your queries as type safe as possible. All of the details of converting your data are handled
by Morphia directly and only rarely do you need to take additional action. The following example illustrates this:

{% highlight java %}
Author read = authorRepo.read(julius.getId());
{% endhighlight %}

# Updating Documents
There are two basic ways to update your data: insert/save a whole Entity or issue an update operation.

## Updating (on the server)
The update method on `Datastore` is used to issue a command to the server to change existing documents. The effects of 
the update command are defined via `UpdateOperations` methods:

{% highlight java %}
UpdateOperations<Author> ops = authorRepo.createOperations().set("name", "Deborah");
final UpdateResults results = authorRepo.update(read, ops);
{% endhighlight %}

Here we are updating the `read` instance obtained from the database and updating the `name` from `Julius` to `Deborah`.

The above executes the update in the database without having to pull in the document. The `UpdateResults` instance 
returned will contain various statistics about the update operation.

## Updating (call create on an updated entity)
Updating data in MongoDB is as simple as updating your Java objects and then calling `datastore.save()` with them 
again:

{% highlight java %}
// read document from database
Author read = authorRepo.read(julius.getId());
read.setName("Abeiku");  // update name
authorRepo.create(read); // update document in database
{% endhighlight %}

# Deleting Documents
After everything else, removes are really quite simple. Removing just needs a query to find and delete the documents 
in question and then tell the `Datastore` to delete them:

{% highlight java %}
Author read = authorRepo.read(julius.getId());
authorRepo.delete(read);
{% endhighlight %}


The full class will look something like this:

file: `src/main/java/com/juliuskrah/morphia/Application.java`

{% highlight java %}
@SpringBootApplication
public class Application {

  public static void main(String[] args) {
    ApplicationContext ctx = SpringApplication.run(Application.class, args);
    BookRepository bookRepo = ctx.getBean(BookRepository.class);
    AuthorRepository authorRepo = ctx.getBean(AuthorRepository.class);
		
    Book ci = new Book("Continous Integration", LocalDate.now());
    bookRepo.create(ci);
		
    Author julius = new Author("Julius");
    julius.setBooks(Stream.of(ci).collect(Collectors.toSet()));
    authorRepo.create(julius);	
		
    Author read = authorRepo.read(julius.getId());
		
    UpdateOperations<Author> ops = authorRepo.createOperations()
      .set("name", "Deborah");
    authorRepo.update(read, ops);
		
    read.setName("Abeiku");
    authorRepo.create(read);
		
    authorRepo.delete(julius);
  }
}
{% endhighlight %}

# Conclusion
In this post we saw the basic mapping between a java object and a MongoDB document. We also saw how to perform basic 
CRUD operations using `Morphia`.  
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.


[DBRef]:                    https://docs.mongodb.com/manual/reference/database-references/#dbrefs
[Gradle]:                   https://gradle.org/
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[MongoDB]:                  https://www.mongodb.com/
[Morphia]:                  http://mongodb.github.io/morphia/
