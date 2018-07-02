---
layout: post
title: "Spring Boot REST API (2) - JPA relationships"
date: 2016-09-25 19:00:00 +0200
author: Martin
description: This is a second post in a series about writing REST API in Spring. This post focuses on JPA and basic entity exposing via REST API.
categories: engineering
tags:
  - Spring-Boot
  - Java
  - REST
---

> Have you heard? 📣 We released a full-feature implemented auth server built on Spring-Boot 2.
> State-of-the-art OAuth2 provider and on top of that - fully open sourced! 🎉🛠
>
> Go check out the [blog post](https://gigsterous.github.io/engineering/2018/06/29/auth-server-example.html) and then [the repository](https://github.com/gigsterous/auth-server) as well!
> Happy hacking! 💻

In the [previous part]({% post_url 2016-09-14-spring-boot-1 %}) of this series, we created a tiny API in Spring Boot which exposes *people* entities over REST. There are usually many resources with (sometimes quite complicated) relationships between each other in a typical web application. In this part, we are going to examine how such relationships can be easily implemented in Spring Boot and Hibernate.

We are going to continue with the project we created last time. If you do not have it and would like to follow this tutorial, you can download it [here](https://github.com/gigsterous/gigy-example/releases/tag/v1) or get the [final result](https://github.com/gigsterous/gigy-example/releases/tag/v2) instead.

The good thing is that no additional dependencies are needed since we already have JPA (which contains Hibernate) in the project so we can jump straight into coding.

## Entities
We will introduce some new entities in our application to make the schema a bit more *complex*. Let's say that our application will provide information about people going to various parties where each person will have some set of skills. The data model will looks as follows:

<p align="center">
  <img src="/assets/2016-09-25-spring-boot-2/diagram.png" alt="Diagram" />
</p>

Let's start by updating our **schema.sql**:

```sql
CREATE TABLE people (
    person_id BIGINT PRIMARY KEY auto_increment,
    name VARCHAR(32),
    age INT,
);

CREATE TABLE skills (
    skill_id BIGINT PRIMARY KEY auto_increment,
    person_id BIGINT REFERENCES people (person_id),
    name VARCHAR(16),
    level VARCHAR(16)
);

CREATE TABLE parties (
    party_id BIGINT PRIMARY KEY auto_increment,
    location VARCHAR(64),
    party_date TIMESTAMP,
);

CREATE TABLE people_parties (
  person_id BIGINT NOT NULL REFERENCES people (person_id),
  party_id BIGINT NOT NULL REFERENCES parties (party_id),
  PRIMARY KEY (person_id, party_id),
);
```

### One-to-Many
The first relationship we will take a look at is a type *one-to-many*. In other words, **one entity** will have a reference to **many entities** of certain type. As described in the diagram above, we can see that each person has a set of skills. In this case, the skill entity is the owner of the relationship since it contains the link to its person.

We can handle this case in JPA by simply putting this reference in the **Person.java**:

```java
@OneToMany(mappedBy = "person")
private Set<Skill> skills = new HashSet<Skill>();
```

We also have to create the **Skill.java** object for the skill entity and put the necessary mappings in there as well:

```java
@Entity
@Table(name = "skills")
public class Skill {

	@Id
	@GeneratedValue(strategy = GenerationType.AUTO)
	@Column(name = "skill_id")
	private long id;

	private String name;

	@Enumerated(EnumType.STRING)
	private Level level;

	@ManyToOne
	@JoinColumn (name="person_id")
	@JsonBackReference
	private Person person;
    
    // getters and setters
}
```

Furthemore, we specified a level for each skill which is an enum value (to have a predefined set of values instead of random "magical" values). Let's go ahead and create it in **Level.java**:

```java

public enum Level {
	GOOD, AWESOME, GODLIKE
}
```

There are several things worth explaining in the code we just created. First of all, as mentioned above, we are going to limit the set of values for each skill level by using an *enum*. There are two ways how to map enums from database to POJOs: 

1. `EnumType.STRING`
2. `EnumType.ORDINAL` 

The first one will require to save strings that match our enum values to the database (eg. 'GOOD') whereas the second one uses integer values matching the position in our enum list (eg. 0 would refer to `GOOD`). There are pros and cons for both approaches but I would personally prefer `EnumType.STRING` since an addition of some additional new value would not corrupt the existing data. In contrast, if we prepend `POOR` to the list and use `EnumType.ORDINAL`, all our database entries will get out of sync.

Second thing to notice is the `@JoinColumn` annotation, where we specify the name of the column that contains the mapped object's foreign key. Since **Skill** is the owner of this relationship, the mapping is done here. In the **Person** object, we simply specify where its foreign key is stored using `@OneToMany(mappedBy = 'person')`.

Lastly, we put `@JsonBackReference` on the person reference in the **Skill** object. When exposing these resources via REST, we need to avoid recursive relationships in JSON and without this annotation (or `@JsonIgnore`) we would create an infinite loop of nested objects in our response. This annotation basically says that a person will not be part of the JSON returned for skill (but each person will contain list of its skills in the response). If the person was a part of the list then the program would fetch the person's skills, which would then make the program fetch the person again and again until we'd get a `StackOverflowException`.

### Many-to-Many
Next relationship we are going to model is many-to-many. People will be linked to various parties that they will/did attend - one person can go to multiple parties and each party can have multiple visitors. In our SQL script, we have to model this relationship using a relationship table called `people_parties`:

```sql
CREATE TABLE parties (
    party_id BIGINT PRIMARY KEY auto_increment,
    location VARCHAR(64),
    party_date TIMESTAMP,
);

CREATE TABLE people_parties (
  person_id BIGINT NOT NULL REFERENCES people (person_id),
  party_id BIGINT NOT NULL REFERENCES parties (party_id),
  PRIMARY KEY (person_id, party_id),
);
```

Again, we have to create the object for a party in **Party.java**:

```java
@Entity
@Table(name = "parties")
public class Party {

	@Id
	@GeneratedValue(strategy = GenerationType.AUTO)
	@Column(name = "party_id")
	private long id;

	private String location;

	@Column(name = "party_date")
	@JsonFormat(pattern = "YYYY-MM-dd")
	private Date date;

	@ManyToMany
	@JoinTable(name = "people_parties", 
		joinColumns = @JoinColumn(name = "party_id", referencedColumnName = "party_id"), 
		inverseJoinColumns = @JoinColumn(name = "person_id", referencedColumnName = "person_id"))
	private Set<Person> people = new HashSet<Person>();
	
	// getters and setters
}
```

and also add this mapping to **Person.java**:

```java
@ManyToMany(cascade = CascadeType.ALL)
@JsonBackReference
@JoinTable(name = "people_parties",
	joinColumns = @JoinColumn(name = "person_id", referencedColumnName = "person_id"),
	inverseJoinColumns = @JoinColumn(name = "party_id", referencedColumnName = "party_id"))
private Set<Party> parties = new HashSet<Party>();
```

The relationship table is mapped to our objects using `@JoinTable` annotation. The name of this table has to be specified along with `joinColumns` (mapping for composite foreign keys) and optionally `inverseJoinColumns` (the foreign key columns of the join table that reference the primary table of the entity that does not own the association).

And finally, if we are going to expose these entities via REST, we need to avoid the infinite recursion in the JSON responses here as well. We'll accomplish that by putting `@JsonBackReference` on one side of the relationship. In this case, all responses for parties will also include people in them but not vice versa.

## Adding controller and testing

The application should now compile but parties are not yet publicly exposed. In order to do that, we first need to create repository for them (same way we did in the [first part]({% post_url 2016-09-14-spring-boot-1 %})):

```java
@Repository
public interface PartyRepository extends CrudRepository<Party, Long> {
	Collection<Party> findAll();
}
```

and also create a controller:

```java
@RestController
@RequestMapping("/parties")
public class PartyController {

	@Autowired
	private PartyRepository partyRepo;

	@RequestMapping(method = RequestMethod.GET)
	public ResponseEntity<Collection<Party>> getParties() {
		return new ResponseEntity<>(partyRepo.findAll(), HttpStatus.OK);
	}

	@RequestMapping(value = "/{id}", method = RequestMethod.GET)
	public ResponseEntity<Party> getParty(@PathVariable long id) {
		Party party = partyRepo.findOne(id);

		if (party != null) {
			return new ResponseEntity<>(partyRepo.findOne(id), HttpStatus.OK);
		} else {
			return new ResponseEntity<>(null, HttpStatus.NOT_FOUND);
		}
	}

	@RequestMapping(method = RequestMethod.POST)
	public ResponseEntity<?> addParty(@RequestBody Party party) {
		return new ResponseEntity<>(partyRepo.save(party), HttpStatus.CREATED);
	}

	@RequestMapping(value = "/{id}", method = RequestMethod.DELETE)
	public ResponseEntity<Void> deletePartyn(@PathVariable long id) {
		partyRepo.delete(id);

		return new ResponseEntity<Void>(HttpStatus.OK);
	}
}
```

Let's also pre-populate our database with some data to test our new service. We can do this in our **data.sql**:

```sql
INSERT INTO skills (skill_id, person_id, name, level) VALUES
	(1, 1, 'Juggling', 'GOOD'),
	(2, 1, 'Dancing', 'AWESOME'),
	(3, 2, 'Juggling', 'AWESOME'),
	(4, 2, 'Story-telling', 'GODLIKE'),
	(5, 3, 'Singing', 'GOOD');

INSERT INTO parties (party_id, location, party_date) VALUES 
	(1, 'Old Folks Club', '2016-09-20'),
	(2, 'Luxury Yacht Party', '2016-12-05');
	
INSERT INTO people_parties (person_id, party_id) VALUES
	(1, 1),
	(1, 2),
	(2, 1),
	(3, 2);
```

Now we are finally ready to run the application and test the outcome at `localhost:8000/gigy/parties` and `localhost:8000/gigy/people`. You should see that people now have set of skills in every response and the responses with parties contain the people attending them.

Notice, that people are part of each party in the JSON response:

```json
{
   "id": 1,
   "location": "Old Folks Club",
   "date": "2016-09-19",
   "people": [
     {
       "id": 2,
       "name": "John",
       "age": 30,
       "skills": [
         {
           "id": 3,
           "name": "Juggling",
           "level": "AWESOME"
         },
         {
           "id": 4,
           "name": "Story-telling",
           "level": "GODLIKE"
         }
      ]
     },
     {
       "id": 1,
       "name": "Peter",
       "age": 25,
       "skills": [
         {
           "id": 1,
           "name": "Juggling",
           "level": "GOOD"
         },
         {
           "id": 2,
           "name": "Dancing",
           "level": "AWESOME"
         }
       ]
     }
   ]
 }
```

It does not work vice versa for the `/people` endpoint since we added the `@JsonBackReference` but we can implement a separate endpoint for accessing parties for a given person:

```java
@RequestMapping(value = "/{id}/parties", method = RequestMethod.GET)
public ResponseEntity<Collection<Party>> getPersonParties(@PathVariable long id) {
	Person person = personRepo.findOne(id);

	if (person != null) {
		return new ResponseEntity<>(person.getParties(), HttpStatus.OK);
	} else {
		return new ResponseEntity<>(null, HttpStatus.NOT_FOUND);
	}
}
```

## Summary
In this part of the Spring Boot REST API series, we covered how to specify one-to-many and many-to-many relationship between our entities in order to demonstrate how to expose some more complex data via REST. The purpose of this post was to give a higher level tutorial for this topic rather than to examine in-depth JPA and Hibernate capabilities. The resulting service we just created can be donwloaded from [Github](https://github.com/gigsterous/gigy-example/releases/tag/v2). Feel free to use it as a starting position for your next project, because [it's published under Apache License 2.0](https://github.com/gigsterous/gigy-example/blob/master/LICENSE).

### Sidenote
Instead of writing our own controllers, we could make use of `spring-boot-data-rest` starter which converts all JPA repositories into a fully functional RESTful API with responses in [HAL/JSON](https://tools.ietf.org/html/draft-kelly-json-hal-08). However, we implemented the controllers manually to demonstrate how to work with them and we will continue to use them in the following parts of this series. If you need a guide for using `spring-boot-data-rest` you can find it in [official resources](https://spring.io/guides/gs/accessing-data-rest/).

