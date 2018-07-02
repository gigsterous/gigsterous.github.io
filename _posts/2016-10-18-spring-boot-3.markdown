---
layout: post
title: "Spring Boot REST API (3) - Unit testing"
date: 2016-10-18 19:00:00 +0200
author: Martin
description: This is a third post in a series about writing REST API in Spring. This post describes how to test different layers of Spring Boot application.
categories: engineering
tags:
  - Spring-Boot
  - Java
  - REST
  - Testing
---

> Have you heard? 📣 We released a full-feature implemented auth server built on Spring-Boot 2.
> State-of-the-art OAuth2 provider and on top of that - fully open sourced! 🎉🛠
>
> Go check out the [blog post](https://gigsterous.github.io/engineering/2018/06/29/auth-server-example.html) and then [the repository](https://github.com/gigsterous/auth-server) as well!
> Happy hacking! 💻

We have already learned how to set up a [simple REST API in Spring Boot]({% post_url 2016-09-14-spring-boot-1 %}) and create a [relational model of our entities]({% post_url 2016-09-25-spring-boot-2 %}). This time, we will take a look at how to write unit tests for all layers in our application.

Spring Boot provides many useful annotations and tools for testing. Things have got significantly easier [starting in Spring Boot 1.4](https://spring.io/blog/2016/04/15/testing-improvements-in-spring-boot-1-4). We already have `spring-boot-starter-test` dependency in our project, so if you don't simply include this in your `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
</dependency>
```

This module includes a lot of useful testing tools - you can see the full list in the [official documentation](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html).

## Testing without Spring context

It is considered a good practice to structure your code in a way that enables you to test individual components of your system separately. Not only that - ideally also without any Spring context. This can be achieved e.g. by constructor injection. Constructor injection allows initializing the components using the `new` keyword rather than having to `@Autowire` them in our unit tests.

We do not have any services in our application yet, so let's create one and then write a unit test for it (or preferably the other way around if you want to do [TDD](https://en.wikipedia.org/wiki/Test-driven_development)). Our service will be a very simple piece of code that takes a person as an input parameter and finds another one (a buddy) using `PersonRepository`. 

First, we will create an interface (it is good practice to use an interface for every service when working with Spring):

```java
public interface BuddyService {
	public Person findBuddy(Person person);
}
```

And now let's crate our service implementation:

```java
@Service
public class DrinkingBuddyService implements BuddyService {
	private PersonRepository repository;
	
	@Autowired
	public DrinkingBuddyService(PersonRepository repository) {
		this.repository = repository;
	}

	/**
	 * Finds and returns person with smallest age difference to the input.
	 */
	@Override
	public Person findBuddy(Person person) {
		Person buddy = null;
		
		for (Person p : repository.findAll()) {
			// do not return same person or person without age
			if (p.getId() == person.getId() || p.getAge() == 0) {
				continue;
			}
			
			// no buddy? Take the first one
			if (buddy == null) {
				buddy = p;
			}
			
			int buddyDiff = Math.abs(person.getAge() - buddy.getAge());
			int currentDiff = Math.abs(person.getAge() - p.getAge());
			
			// found someone closer to this person's age
			if (currentDiff < buddyDiff) {
				buddy = p;
			}
		}
		
		return buddy;
	}
}
```

This code logic might not be the best you'll ever see, but it is easy to understand and sufficient for demonstrating how a test can be written for Spring services. Furthermore, notice the `@Service` annotation in our class. This annotation registers our class as a Spring component and lets us inject it into Spring context using the following construct (same as our repositories):

```java
@Autowire
BuddyService buddyService;
```

We also inject our `PersonRepository` to our service in constructor, because we want to be able to write a test that will inject it using a classic constructor (hence, without the Spring context). 

Let's take a look at what our test could look like:

```java
public class DrinkingBuddyServiceTest {
	
	private PersonRepository repository;
	private DrinkingBuddyService service;
	
	@Before
	public void prepare() {
		repository = mock(PersonRepository.class);
		service = new DrinkingBuddyService(repository);
	}
	
	@Test
	public void findBuddyTest() {
	    List<Person> people = new ArrayList<Person>();
	    
	    Person p1 = new Person();
	    p1.setId(1l);
	    p1.setName("John");
	    p1.setAge(25);
	    people.add(p1);
	    
	    Person p2 = new Person();
	    p2.setId(2l);
	    p2.setName("Marry");
	    p2.setAge(22);
	    people.add(p2);
	    
	    Person p3 = new Person();
	    p3.setId(1l);
	    p3.setName("Peter");
	    p3.setAge(35);
	    people.add(p3);
	    
	    when(repository.findAll()).thenReturn(people);
	    
	    assertEquals(service.findBuddy(p1), p2);
	}
}
```

One important concept to keep in mind when writing unit tests is that we want to isolate the tested class and test only one layer of the application at a time. That's why we mock the repository and prepare a predefined response for `findAll()` to be returned when it gets called. This is achieved using [Mockito](http://mockito.org/) which is part of the Spring test module. All we have to do is to call the method from `DrinkingBuddyService` we are testing and [assert](http://joel-costigliola.github.io/assertj/) that the response is as expected.

## Context slicing

Testing without Spring context is fast and useful but it cannot be done that way in all situations. Sometimes, we need to have a `SpringContext` to test some parts of our application. Starting in Spring Boot version 1.4, we can make a use of [slicing](https://spring.io/blog/2016/04/15/testing-improvements-in-spring-boot-1-4) feature, which launches only the part of Spring context that we need.

You can, of course, still launch the whole Spring context using `@SpringBootTest` annotation when performing integration testing. We will not go into details on this topic, though.

### Testing controllers

To test our controllers, we need to launch the `WebMvc` context slice. Ideally, we should only load the single controller we are going to test instead of the whole web layer. This can be done easily with `@WebMvcTest` annotation:

```java
@RunWith(SpringRunner.class)
@WebMvcTest(PersonController.class)
public class PersonControllerTest {

	@Autowired
	private MockMvc mvc;

	@MockBean
	private PersonRepository personRepo;

	@Test
	public void getPersonTest() throws Exception {
	    Person person = new Person();
	    person.setId(1l);
		person.setName("John");
		person.setAge(25);
	
		given(personRepo.findOne(1l)).willReturn(person);
		mvc.perform(get("/people/1").accept(MediaType.APPLICATION_JSON_VALUE)).andExpect(status().isOk())
				.andExpect(jsonPath("$.id", is(1)))
				.andExpect(jsonPath("$.name", is("John")))
				.andExpect(jsonPath("$.age", is(25)));
	}
	
	@Test
	public void personNotFoundTest() throws Exception {
		mvc.perform(get("/people/2").accept(MediaType.APPLICATION_JSON_VALUE)).andExpect(status().isNotFound());
	}
}
```

Let's stop for a while and see what we just did here. By adding `@RunWith(SpringRunner.class)` to our test class, we tell JUnit to start running using Spring's testing support. Furthermore, `@WebMvc(PersonController.class)` specifies that `WebMvc` context will be launched only with `PersonController` class initialized.

We need to autowire the `MockMvc` bean, because we need to mock some requests for our endpoint in our test. Finally, `PersonRepository` needs to be mocked and provided as a bean to our context, because it is autowired in the tested `PersonController` and Spring would complain about this bean being missing otherwise (Spring loads everything that is needed in order to initialize this controller into the context). Why mock it and not load it into the context as well? Well, we are testing only one layer (controller, in this example) at a time, remember?

The rest of this test is very straight-forward. We make request to our endpoint using `MockMvc` and examine the response. One thing that is probably worth pointing out is the `jsonPath` method which comes from [Hamcrest](http://hamcrest.org/JavaHamcrest/) and lets us check various parts of the JSON response. 

## Testing JPA repositories

Another thing we can test are our repositories. There is another useful context slice just for this layer: `@DataJpaTest`. Let's take a look at what this test could look like:

```java
@RunWith(SpringRunner.class)
@DataJpaTest
public class PersonRepositoryTest {

    @Autowired
    private PersonRepository repository;

    @Test
    public void repositorySavesPerson() {
        Person person = new Person();
        person.setName("John");
        person.setAge(25);
        
        Person result = repository.save(person);
        
        assertEquals(result.getName(), "John");
        assertEquals(result.getAge(), 25);
    }
}
```

Please note, that this test is very simple and does not test `EntityManager` explicitly. It is here just to demonstrate the basic test skeleton and custom JPA slice, which provides the data layer to the context.

## More context slices

Our application is still quite simple so we will not be showing all the available context slices but they are worth mentioning nonetheless:

* `@JsonTest` - JSON serialization and deserialization
* `@RestClientTest` - REST clients testing
* `@AutoConfigureRestDocs` - Spring REST Docs testing

## Summary

In this part of Spring Boot REST API series, we learned how to write simple unit tests for various layers of our application and explored how Spring context slicing can be used to make our tests easier and faster to run. As always, up-to-date code of the example application can be downloaded from [GitHub](https://github.com/gigsterous/gigy-example/releases/tag/v3).
