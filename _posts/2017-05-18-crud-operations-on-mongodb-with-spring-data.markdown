---
layout:     post
title:      CRUD Operations on MongoDB with Spring Data
date:       2017-05-18 22:57:07 +0000
categories: blog
tags:       java gradle mongodb spring-data spring-boot
section:    blog
author:     juliuskrah
repo:       morphia-example/tree/spring-data
---
>  Spring Data for [`MongoDB`][MongoDB]{:target="_blank"} is part of the umbrella Spring Data project which aims to 
   provide a familiar and consistent Spring-based programming model for for new datastores while retaining 
   store-specific features and capabilities.  
   The Spring Data MongoDB project provides integration with the MongoDB document database. Key functional areas of 
   Spring Data MongoDB are a POJO centric model for interacting with a MongoDB DBCollection and easily writing a 
   Repository style data access layer.

# Introduction
In this post we will learn how to implement basic Create, Read, Update and Delete (CRUD) operations using 
`Spring Data` on MongoDB. If you would like to learn how to perform CRUD operations with `Spring Data` on 
[relational]({% post_url 2017-03-22-crud-operations-with-spring-data-jpa %}) databases, you can have a look at my 
previous blog post. If you are interested in an alternative approach to persisting data in MongoDB you can
head over to my [`Morphia`]({% post_url 2017-05-14-crud-operations-with-morphia %}) post.  
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
|  |  |  |  |__data/    
|  |  |  |  |  |__Application.java
|  |  |  |  |  |__entity/
|  |  |  |  |  |  |__Author.java
|  |  |  |  |  |  |__Book.java
|  |  |  |  |  |__repository/
|  |  |  |  |  |  |__AuthorRepository.java
|  |  |  |  |  |  |__BookRepository.java
|  |  |__resources/
|  |  |  |__application.yaml
|__build.gradle
```

## Project Dependencies
Before we can use `Spring Data` to persist into MongoDB, we need to add the dependency in our gradle buildscript.

file: `build.gradle`

{% highlight gradle %}
...
dependencies {
  compile 'org.springframework.boot:spring-boot-starter-data-mongodb'
}
{% endhighlight %}

The above dependency will pull in `Spring Boot` and `Spring Data` which we need for this blog post.

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
  baseName = 'spring-data'
  version = '0.0.1-SNAPSHOT'
}
sourceCompatibility = 1.8
targetCompatibility = 1.8

repositories {
  mavenCentral()
}

dependencies {
  compile 'org.springframework.boot:spring-boot-starter-data-mongodb'
}
{% endhighlight %}

# Configure Spring Application
Let us start with setting up our Spring-Boot application

file: `src/main/java/com/juliuskrah/data/Application.java`

{% highlight java %}
@SpringBootApplication
public class Application {

  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }
}
{% endhighlight %}

With Spring Boot most of the configuration options are handled by the framework. We will just override the MongoDB 
database options in an external configuration file.

file: `src\main\resources\application.yaml`

{% highlight yaml %}
spring:
  data:
    mongodb:
      database: spring_data

# Optional [Add these only if your MongoDB instance is secured]
      authentication-database: some-database
      username: some-user
      password: secret
{% endhighlight %}

# Object Document Mapping
With all the basic requirements out of the way, we can start mapping our Java Objects to MongoDB documents using
an `Author` to `Book` example. An author can write many books, and a book can have multiple authors. We will keep
this simple by focusing on an author with many books and not the inverse relationship.

Rich mapping support is provided by the `MappingMongoConverter`. `MappingMongoConverter` has a rich metadata model that 
provides a full feature set of functionality to map domain objects to MongoDB documents.The mapping metadata model is
populated using annotations on your domain objects. However, the infrastructure is not limited to using annotations as
the only source of metadata information. The `MappingMongoConverter` also allows you to map objects to documents 
without providing any additional metadata, by following a set of conventions.

## Convention based Mapping
`MappingMongoConverter` has a few conventions for mapping objects to documents when no additional mapping metadata is
provided. The conventions are:

- The short Java class name is mapped to the collection name in the following manner. The class 
  `com.juliuskrah.data.entity.Author` maps to `author` collection name.  
- All nested objects are stored as nested objects in the document and not as [`DBRefs`][DBRef]{:target="_blank"}
- The converter will use any Spring Converters registered with it to override the default mapping of object properties'
  to document field/values.  
- The fields of an object are used to convert to and from fields in the document. Public JavaBean properties are not 
  used.
- You can have a single non-zero argument constructor whose constructor argument names match top level field names of 
  document, that constructor will be used. Otherwise the zero arg constructor will be used. if there is more than one
  non-zero argument constructor an exception will be thrown.

## Metadata based Mapping
To take full advantage of the object mapping functionality inside the Spring Data/MongoDB support, you should annotate
your mapped objects with the `@Document` annotation. Although it is not necessary for the mapping framework to have 
this annotation (your POJOs will be mapped correctly, even without any annotations), it allows the classpath scanner to
find and pre-process your domain objects to extract the necessary metadata. If you don’t use this annotation, your 
application will take a slight performance hit the first time you store a domain object because the mapping framework 
needs to build up its internal metadata model so it knows about the properties of your domain object and how to persist
them.

file: `src/main/java/com/juliuskrah/data/entity/Book.java`

{% highlight java %}
@Document(collection = "books")
public class Book {
  @Id
  private ObjectId id;
  private String title;
  @Field("published")
  private LocalDate publicationDate;

  // No args Constructor
  public Book(String title, LocalDate publicationDate) {
    this.title = title;
    this.publicationDate = publicationDate;
  }

  // Getters and Setters omitted for brevity
}
{% endhighlight %}

file: `src/main/java/com/juliuskrah/data/entity/Author.java`

{% highlight java %}
@Document(collection = "authors")
public class Author {
  @Id
  private ObjectId id;
  @Indexed
  private String name;
  @DBRef
  private Set<Book> books;

  // No args Constructor
  public Author(String name) {
    this.name = name;
  }

  // Getters and Setters omitted for brevity
}
{% endhighlight %}

> The `@Id` annotation tells the mapper which property you want to use for the MongoDB `_id` property and the 
  `@Indexed` annotation tells the mapping framework to call `createIndex(…)` on that property of your document, making searches faster. 

> Automatic index creation is only done for types annotated with `@Document`

The MappingMongoConverter can use metadata to drive the mapping of objects to documents. An overview of the annotations
used above is provided below:

- `@Id` - applied at the field level to mark the field used for identity purpose.
- `@Document` - applied at the class level to indicate this class is a candidate for mapping to the database. You can 
  specify the name of the collection where the database will be stored.
- `@DBRef` - applied at the field to indicate it is to be stored using a `com.mongodb.DBRef`.
- `@Indexed` - applied at the field level to describe how to index the field.
- `@Field` - applied at the field level and describes the name of the field as it will be represented in the MongoDB
  BSON document thus allowing the name to be different than the field name of the class.

# Creating the CRUD repository
We will create a simple repository class to take care of our CRUD operations.

file: `src/main/java/com/juliuskrah/data/repository/AuthorRepository.java`

{% highlight java %}
public interface AuthorRepository extends Repository<Author, ObjectId> {
  Optional<Author> findOne(ObjectId id);
  Optional<Author> save(Author entity);
  void delete(Author entity);
}
{% endhighlight %}

file: `src/main/java/com/juliuskrah/data/repository/BookRepository.java`

{% highlight java %}
public interface BookRepository extends Repository<Book, ObjectId> {
  Optional<Book> findOne(ObjectId id);
  Optional<Book> save(Book entity);
  void delete(Book entity);
}
{% endhighlight %}

The above interfaces extend `Repository`. `Repository` is the central repository marker interface. It captures the 
domain type to manage as well as the domain type's id type. General purpose is to hold type information as well as 
being able to discover interfaces that extend it during classpath scanning for easy Spring bean creation.  
Domain repositories extending this interface can selectively expose CRUD methods by simply declaring methods of the 
same signature as those declared in `CrudRepository`.

# Creating and Updating Documents
For the most part, you treat your Java objects just like you normally would. When you’re ready to write an object to 
the database, it’s as simple as this:

{% highlight java %}
BookRepository bookRepo =     // get instance;
AuthorRepository authorRepo = // get instance;
		
Book ci = new Book("Continous Integration", LocalDate.now());
// id will be generated after save
bookRepo.save(ci);
		
Author julius = new Author("Julius");
julius.setBooks(Stream.of(ci).collect(Collectors.toSet()));
authorRepo.save(julius);	
{% endhighlight %}

To perform an update, retrieve the `Book` or `Author` instance from the database, mutate the fields and call `save`
on it again. 

# Reading Documents
Spring Data attempts to make your queries as type safe as possible. All of the details of converting your data are
handled by Spring Data directly and only rarely do you need to take additional action. The following example 
illustrates this:

{% highlight java %}
Author read = authorRepo.findOne(julius.getId());
{% endhighlight %}

# Deleting Documents
After everything else, removes are really quite simple. Removing just needs a query to find and delete the documents 
in question and then tell the `Repository` to delete them:

{% highlight java %}
Author read = authorRepo.findOne(julius.getId());
authorRepo.delete(read);
{% endhighlight %}


The full class will look something like this:

file: `src/main/java/com/juliuskrah/data/Application.java`

{% highlight java %}
@SpringBootApplication
public class Application {

  public static void main(String[] args) {
    ApplicationContext ctx = SpringApplication.run(Application.class, args);
    BookRepository bookRepo = ctx.getBean(BookRepository.class);
    AuthorRepository authorRepo = ctx.getBean(AuthorRepository.class);
		
    Book ci = new Book("Continous Integration", LocalDate.now());
    bookRepo.save(ci);
		
    Author julius = new Author("Julius");
    julius.setBooks(Stream.of(ci).collect(Collectors.toSet()));
    authorRepo.save(julius);
		
    Optional<Author> read = authorRepo.findOne(julius.getId());
		
    read.get().setName("Abeiku");
    authorRepo.save(read.get());
		
    authorRepo.delete(julius);
  }
}
{% endhighlight %}

# Conclusion
In this post we saw the basic mapping between a java object and a MongoDB document. We also saw how to perform basic 
CRUD operations using `Spring Data`.  
As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.

[DBRef]:                    https://docs.mongodb.com/manual/reference/database-references/#dbrefs
[Gradle]:                   https://gradle.org/
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[MongoDB]:                  https://www.mongodb.com/
[Morphia]:                  http://mongodb.github.io/morphia/