---
layout: post
title: Spring Data REST and projections
date: 2018-01-26 15:00:00 +0200
author: Martin
description: Let's take a look at some useful tricks with Spring Data REST and projections.
categories: engineering
tags:
  - Spring-Boot
  - Java
  - Spring Data REST
---

> Have you heard? 📣 We released a full-feature implemented auth server built on Spring-Boot 2.
> State-of-the-art OAuth2 provider and on top of that - fully open sourced! 🎉🛠
>
> Go check out the [blog post](https://gigsterous.github.io/engineering/2018/06/29/auth-server-example.html) and then [the repository](https://github.com/gigsterous/auth-server) as well!
> Happy hacking! 💻

Using [Spring Data REST](https://projects.spring.io/spring-data-rest/) can get your API up and running in minutes and save you tons of work and coding...until you have a special use-case requiring you to spend a whole afternoon crawling the web for how to achieve that one missing piece of functionality.

In this post, I would like to share with you how to use [projections](https://docs.spring.io/spring-data/rest/docs/current/reference/html/#projections-excerpts). My use-case was quite simple: 

Let's have two entities: **Cycle** and **Record**.

Cycle contains list of records:

```java
@Entity
@Table(name="cycles")
public class Cycle {

  @Id
  @GeneratedValue(strategy = GenerationType.AUTO)
  private long id;
  
  ...
  
  @OneToMany(mappedBy = "cycle", cascade = CascadeType.ALL)
  private List<Record> records;
}

@Entity
@Table(name = "records")
public class Record {

  @Id
  @GeneratedValue(strategy = GenerationType.AUTO)
  private long id;

  ...

  @ManyToOne
  @JoinColumn(name = "cycle_id")
  private Cycle cycle;
}
```

Both these entities have corresponding repositories:

```java
@RepositoryRestResource
public interface CycleRepository extends PagingAndSortingRepository<Cycle, Long> {
}

@RepositoryRestResource
public interface RecordRepository extends JpaRepository<Record, Long> {
}
```

Since we are using Spring Data REST, these repositories are automatically exposed via REST API in HAL/JSON format. This is what a sample response for `GET /cycles/1` looks like by default:

```json
{
    "id": 1,
    ...
    "_links": {
        "self": {
            "href": "http://localhost:8008/cycles/1"
        },
        "records": {
            "href": "http://localhost:8008/cycles/1/records"
        }
    }
} 
```

But what if we want to get all record objects as part of the cycle instead of providing just a HAL link? The answer is `@Projection`! Projections allow us to alter the view model of our data. Following the official documentation, we can define what we want the response to look like:

```java
@Projection(name = "inlineRecords", types = { Cycle.class })
public interface InlineRecords {

  long getId();

  // other getters
  ...

  List<Record> getRecords();
}
```

Adding this projection to our `CycleRepository` is quite easy:

`@RepositoryRestResource(excerptProjection = InlineRecords.class)`

When sending a request to fetch a specific cycle, we can specify which projection to use: `GET /cycles/1?projection=inlineRecords`:

```json
{
    "id": 1,
    ...
    "_records": [
        .. records
    ],
    "_links": {
       ... links
    }
} 
```

The only problem is that, by default, Spring Data REST applies these projections to all `_embedded` data so when you request all cycles using `GET /cycles`, our inline projection will be used. That means that we will have all our records unwrapped in every cycle which could potentially be a very long response.

Fortunately, there is an alternative way how to register a projection and that is by using `RepositoryRestConfigurerAdapter`:

```java
@Configuration
public class RepositoryConfig extends RepositoryRestConfigurerAdapter {

  @Override
  public void configureRepositoryRestConfiguration(RepositoryRestConfiguration config) {
    config.getProjectionConfiguration().addProjection(InlineRecords.class);
  }
}
```

This way, we can choose when the projection should be applied:
- `GET /cycles`
- `GET /cycles?projection=inlineRecords`
- `GET /cycles/{id}`
- `GET /cycles/{id}?projection=inlineRecords`

It is not hundred percent clear from the documentation what is the default behavior when considering different ways of using projections so I hope that this post might have saved you some frenetic Googling when you found yourself in need of this functionality.

