---
layout: post
title: "Spring Boot REST API (1) - Project setup"
date: 2016-09-14 19:00:00 +0200
author: Martin
description: This is a first post in a series about writing REST API in Spring. This post focuses on the very first project setup.
categories: engineering
tags:
  - Spring-Boot
  - Java
  - REST
---

This is the first part of my series on building a REST API with [Spring Boot](http://projects.spring.io/spring-boot/). Although there are many tutorials on this manner on the internet, my goal is to provide a step-by-step guide for designing the web application from scratch and gradually add more functionality to it in the next parts of this series.

In this tutorial, we are going to start simple and create a small Spring Boot app exposing one entity over REST. We will also setup a test environment with in-memory database to prepopulate our API with some data.

You can either follow the tutorial on your own or download the source code used in this blog post from [Github](https://github.com/gigsterous/gigy-example/releases/tag/v1).

## Starting the project
First, create a Spring Boot app in our IDE (if supported) or download a basic skeleton from [SpringInitializr](https://start.spring.io/) and import it. We will be using Spring Boot 1.4 and Maven in this tutorial.

Open your `pom.xml` and add the following dependencies:

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>com.h2database</groupId>
	<artifactId>h2</artifactId>
	<scope>runtime</scope>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-test</artifactId>
	<scope>test</scope>
</dependency>
```
- `spring-boot-starter-data-jpa` provides easy access to relational database 
- `spring-boot-starter-web` is needed for building web (and RESTful) applicaations, contains embedded Tomcat as a servlet container
- `spring-boot-starter-test` includes various test libraries including JUnit, Mockito, etc.
- `h2` provides databse engine for our application

## Data model
In this tutorial, we will create a simple web application with one entity - people - exposed via REST. Let's start by creating the class representing our model:

```java
@Entity
@Table(name = "people")
public class Person {

	@Id
	@GeneratedValue(strategy = GenerationType.AUTO)
	@Column(name = "person_id")
	private long id;

	private String name;
	private int age;

	public Person() {
		// empty constuctor for Hibernate
	}

	public long getId() {
		return id;
	}

	public void setId(long id) {
		this.id = id;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}
}
```

We will also create the SQL script for creating the table for our entity and prefill some data in it. Spring Boot lets us do this easily by placing `schema.sql` and `data.sql` scripts in the root of the resources directory (next to `application.properties`). By default, in-memory H2 database will be used. We could change this and hook our service to a real database by specifying it in `application.properties`.

Create these scripts and put the following code into `schema.sql`:

```sql
CREATE TABLE people (
    person_id BIGINT PRIMARY KEY auto_increment,
    name VARCHAR(32),
    age INT,
);
```

Lets also prefill some data to our database in `data.sql`:

```sql
INSERT INTO people (person_id, name, age) VALUES 
	('1', 'Peter', 25),
	('2', 'John', 30),
	('3', 'Katie', 18);
```

Finally, we will do some modifications in our `application.properties`. Te following changes are not necessary to run this project but will demonstrate how we can customize the application entry point and port:

```
server.port=8000
server.contextPath=/gigy
```

## Creating REST controller
Finally, lets create a controller for accessing the data via REST. Creating controllers can be done easily in Spring by placing `@RestController` on a class. `@RequestMapping` annotation is then used for specifying the endpoint properties including path, Http method, content type etc.

```java
@RestController
@RequestMapping("/people")
public class PersonController {

	@Autowired
	private PersonRepository personRepo;

	@RequestMapping(method = RequestMethod.GET)
	public ResponseEntity<Collection<Person>> getPeople() {
		return new ResponseEntity<>(personRepo.findAll(), HttpStatus.OK);
	}

	@RequestMapping(value = "/{id}", method = RequestMethod.GET)
	public ResponseEntity<Person> getPerson(@PathVariable long id) {
		Person person = personRepo.findOne(id);

		if (person != null) {
			return new ResponseEntity<>(personRepo.findOne(id), HttpStatus.OK);
		} else {
			return new ResponseEntity<>(null, HttpStatus.NOT_FOUND);
		}
	}

	@RequestMapping(method = RequestMethod.POST)
	public ResponseEntity<?> addPerson(@RequestBody Person person) {
		return new ResponseEntity<>(personRepo.save(person), HttpStatus.CREATED);
	}

	@RequestMapping(value = "/{id}", method = RequestMethod.DELETE)
	public ResponseEntity<Void> deletePerson(@PathVariable long id) {
		personRepo.delete(id);
		
		return new ResponseEntity<Void>(HttpStatus.OK);
	}
}
```

## Testing the service
We are ready to test the outcome yet. Run the application and try accessing, modifying and deleting data by accessing these REST endpoints:

**GET**
```
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X GET http://localhost:8000/gigy/people
```
should return all the entities in our database in JSON format.

**POST**
```
curl -H "Content-Type: application/json" -X POST -d '{"name":"Mark","age":40}' http://localhost:8000/gigy/people
```

**DELETE**
```
curl -X DELETE http://localhost:8000/gigy/people/2
```

## Summary
In this tutorial, we covered a basic project setup with in-memory database accessed by JPA and accessin its data via REST. This project will be used as a base for the next parts of this series which will cover more topics about building a RESTful API.
