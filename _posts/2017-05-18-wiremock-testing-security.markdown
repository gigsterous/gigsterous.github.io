---
layout: post
title: "Integration testing OAuth2 security with Spring Boot and WireMock"
date: 2017-05-18 19:00:00 +0200
author: Martin
description: This is a post explaining how to mock Spring OAuth2 security in integration tests with Wiremock.
categories: engineering
tags:
  - Spring-Boot
  - Java
  - Testing
  - Wiremock
---

> Have you heard? 📣 We released a full-feature implemented auth server built on Spring-Boot 2.
> State-of-the-art OAuth2 provider and on top of that - fully open sourced! 🎉🛠
>
> Go check out the [blog post](https://gigsterous.github.io/engineering/2018/06/29/auth-server-example.html) and then [the repository](https://github.com/gigsterous/auth-server) as well!
> Happy hacking! 💻

Recently, I was working with a couple of Spring Boot services that authenticate against a separate auth server using `spring-security-oauth2` package. All that is needed to hook the services to the server is placing `security.oauth2.resource.userInfoUri=http://localhost:8080/principal` in the `application.properties` and annotating one of the configuration classes with `@EnableResourceServer`. That's it - now everytime someone makes a request to your service, a request will be made to the auth server which will check the validity of the Bearer token and return a principal with user information from the specified endpoint.

The challenging part was writing the integration tests (starting with the controller layer and all the filters before including security).

Let's say you have an endpoint `/hello` which returns some String. Your integration test could look as follows:

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class HelloControllerIT {

    @Autowired
    private TestRestTemplate testRestTemplate;
    
    @Test
    public void testHelloEndpoint() {
        HttpHeaders headers = new HttpHeaders();
        // place your headers here
        
        final ResponseEntity<String> response = testRestTemplate.exchange("/hello", HttpMethod.GET), new HttpEntity<>(headers), String.class);
                
        assertThat(response.getStatusCode(), is(HttpStatus.OK));
        assertThat(response.getBody(), is("Hello, World!"));
    }
}
```

In case your service is secured with OAuth2, you will likely get a `401 - Unauthorized` response in this test. There are multiple ways how to get around this. Some of them are very straight forward (turning off the security in the test profile) and some of them can get pretty complex. Spring offers some ways for mocking OAuth2 security in the tests but it can be quite challenging to make it work.

## WireMock
Here is where I find [WireMock](http://wiremock.org/) extremely helpful. This powerful mocking engine enables you to mock responses from an external server and it integrates beautifully with JUnit. Of course, it can do much more (like running as a standalone server and capturing/replaying server responses or acting as a proxy) but we will focus on how to use it for mocking security in this short tutorial.

First of all, add WireMock as a dependency to your project:

```xml
<dependency>
    <groupId>com.github.tomakehurst</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <version>2.6.0</version>
    <scope>test</scope>
</dependency>
```

Now we only need to do a couple of modifications to our test class and everything should work like a charm. The idea is, that we will mock the response containing the principal from the auth server when running our tests. Spring security will automatically see that the principal object was fetched successfully and will inject it into the security context.

First of all, initialize `WireMockRule`

```java
@Rule
public WireMockRule wireMockRule = new WireMockRule();
```

and configure your endpoint stub to return any principal you need.

```java
stubFor(get("/principal")
                .willReturn(okJson("{\"username\":\"john\",\"authorities\":[{\"authority\":\"ROLE_USER\"}]}")));
```

By default, WireMock is running on `localhost:8080` but you can change this in your test configuration.

This is what the whole test class looks with WireMock:

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class HelloControllerIT {

    @Autowired
    private TestRestTemplate testRestTemplate;
    
    @Rule
    public WireMockRule wireMockRule = new WireMockRule();
    
    @Test
    public void testHelloEndpoint() {
        stubFor(get("/principal")
                .willReturn(okJson("{\"username\":\"john\",\"authorities\":[{\"authority\":\"ROLE_USER\"}]}")));
    
        HttpHeaders headers = new HttpHeaders();
        headers.add(HttpHeaders.AUTHORIZATION, "Bearer xyz123");
        
        final ResponseEntity<String> response = testRestTemplate.exchange("/hello", HttpMethod.GET), new HttpEntity<>(headers), String.class);
                
        assertThat(response.getStatusCode(), is(HttpStatus.OK));
        assertThat(response.getBody(), is("Hello, World!"));
    }
}
```

Don't forget to put an authorization header with some bearer token (it could be anything since we are mocking it in our test) in your request headers. Otherwise, Spring will not try to fetch the principal if it is missing.

Your test should be passing now if you did everything correctly. Congratulations! As I mentioned earlier, WireMock is much more powerful than this so do take the time and take a look at its [website](http://wiremock.org/) or [GitHub repository](https://github.com/tomakehurst/wiremock). 
