---
layout: post
title: "Spring Boot REST API (4) - Security with OAuth2"
date: 2017-03-01 20:00:00 +0200
author: Martin
description: This post describes how to implement OAuth2 security in an existing REST API.
categories: engineering
tags:
  - Spring-Boot
  - Java
  - REST
  - Security
---

In this post, we will continue with development of our simple REST API. This time, the focus will be on security. It is not an uncommon scenario that certain resources (or all of them) need to be protected and accessible only by registered users. There are multiple ways how to secure REST APIs and we cannot say that one is better than the other - it all depends on what we are trying to achieve.

[OAuth2](https://oauth.net/2/) is a protocol (or authorization framework, as I prefer to refer to it) that describes a **stateless** authorization (that means we don't need to maintain sessions between clients and our server). Before we jump to code examples, let me briefly explain how OAuth2 works. I will not go into exhaustive details since this has been already described [elsewhere](http://tutorials.jenkov.com/oauth2/index.html).

This protocol allows third party clients to access protected resources on behalf of the resource owner. There are four basic roles in OAuth2:
* **Resource owner** - the owner of the resource - this is pretty self-explanatory :-)
* **Resource server** - the server hosting all the protected resources
* **Client** - the application accessing the resource server
* **Authorization server** - the server that handles issuing access tokens to clients. This could be the same server as the resource server

Furthermore, there are two types of tokens:
* **access token**, which usually has limited lifetime and enables the client to access protected resources by including this token in the request header
* **refresh token** with longer lifetime used to get a new access token once it expires (without the need of sending credentials to the server again)

It is important to note, that OAuth2 should be used with HTTPS because it requires the client to exchange sensitive information with the server (tokens or credentials).

Clients need to be registered with the authorization server in order to receive their client-id and client-secret which are later used when requesting the access tokens. Each token has a **scope** which is defined by the user when communicating with the authorization server (e.g. the user authorizes the client application to access certain resources on the resource server).

There are four different **grant types** defined by OAuth2. These grant types define interactions between the client and the auth/resource server. More detailed information can be found [here](https://alexbilbie.com/guide-to-oauth-2-grants/).
* **Authorisation code** - redirection-based flow for confidential client, the client communicates with the server via user-agent (web browser etc.), typical for web servers
* **Implicit** - typically used with public clients running in a web browser using a scripting language, does not contain refresh tokens, this grant does not contain authentication and relies on redirection URI specified during client registration to the auth server
* **Resource owner password credentials** - used with trusted clients (e.g. clients written by the same company that owns the auth server), user credentials are passed to the client and then to the auth server and exchanged for access and refresh tokens
* **Client credentials** - used when the client itself is the resource owner (one client does not operate with multiple users), client credentials are exchanged directly for the tokens

## Spring Boot and OAuth2
Now that we have some grasp on the theory, let's jump to our example. We will take our API from our last post (you can download the source code from [github](https://github.com/gigsterous/gigy-example/releases/tag/v3)) and implement our own OAuth2 security. There will be multiple users in our system, each with privileges to edit and delete only their own resources. We will also demonstrate OAuth2 using the **resource owner password credentials grant** since it best matches our use case. Furthermore, for the sake of simplicity, we will have the resource server and auth server together as part of the same application.

Let us start by adding the necessary dependencies:

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-security</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.security.oauth</groupId>
	<artifactId>spring-security-oauth2</artifactId>
	</dependency>
</dependencies>
```

By default, if we do not configure Spring security, our API will be secured by basic auth using username and password generated during application startup. However, we are going to configure our OAuth setup and add some test users to our in-memory database to see how it works.

First of all, add `@EnableResourceServer` to the main class:

```java
@SpringBootApplication
@EnableResourceServer
public class App {
	public static void main(String[] args) {
		SpringApplication.run(App.class, args);
	}
}
```

`@EnableResourceServer` will turn our application into a resource server (enables Spring Security filter to authenticate requests via an incoming OAuth2 token). This secures everything in the server except for the oauth endpoints, e.g. `/oauth/authorize`.

Next, we need to create an entity representing a user. Spring security uses an interface called `UserDetails` to encapsulate the user information in Authentication object:

```java
@Entity
@Table(name = "users")
public class User implements UserDetails {
	
	static final long serialVersionUID = 1L;
	
	@Id
	@GeneratedValue(strategy = GenerationType.AUTO)
	@Column(name = "user_id", nullable = false, updatable = false)
	private Long id;
	
	@Column(name = "username", nullable = false, unique = true)
	private String username;
	
	@Column(name = "password", nullable = false)
	private String password;
	
	@Column(name = "enabled", nullable = false)
	private boolean enabled;

	@Override
	public Collection<? extends GrantedAuthority> getAuthorities() {
		List<GrantedAuthority> authorities = new ArrayList<GrantedAuthority>();
		
		return authorities;
	}

	@Override
	public boolean isAccountNonExpired() {
		return true;
	}

	@Override
	public boolean isAccountNonLocked() {
		// we never lock accounts
		return true;
	}

	@Override
	public boolean isCredentialsNonExpired() {
		// credentials never expire
		return true;
	}

	@Override
	public boolean isEnabled() {
		return enabled;
	}

	@Override
	public String getPassword() {
		return password;
	}

	@Override
	public String getUsername() {
		return username;
	}
}
```

We can now add the SQL table definition and add some sample data into our SQL migration scripts:

```sql
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY auto_increment,
    username VARCHAR(128) UNIQUE,
    password VARCHAR(256),
    enabled BOOL,
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

```sql
INSERT INTO users (user_id, username, password, enabled) VALUES 
	('1', 'peter@example.com', '$2a$10$D4OLKI6yy68crm.3imC9X.P2xqKHs5TloWUcr6z5XdOqnTrAK84ri', true),
	('2', 'john@example.com', '$2a$10$D4OLKI6yy68crm.3imC9X.P2xqKHs5TloWUcr6z5XdOqnTrAK84ri', true),
	('3', 'katie@example.com', '$2a$10$D4OLKI6yy68crm.3imC9X.P2xqKHs5TloWUcr6z5XdOqnTrAK84ri', true);
```

Notice, that we store the password in an encrypted form. Never, ever store your users' passwords in plain text. Period (Oh, and also, don't use SHA-1). It would be handy to be able to retrieve users from the database as well. For this, we will need a JPA repository and a `UserDetailsService` which will enable us to pull these user details in the security context.

```java
public interface UserRepository extends JpaRepository<User, Long> {
	User findOneByUsername(String username);
}
```

```java
@Service("userDetailsService")
public class UserService implements UserDetailsService {
	
	@Autowired
	private UserRepository userRepository;

	@Override
	public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
		return userRepository.findOneByUsername(username);
	}
}
```

Notice that `UserDetailsService` needs to implement only one method - `loadUserByUsername`. This method is used to determine which user is logged in, asuming that every user has a unique username.

Now that we have the user infrastructure in place, we can hook it to our OAuth2 configuration:

```java
@Configuration
@EnableAuthorizationServer
public class OAuth2Config extends AuthorizationServerConfigurerAdapter {

	@Autowired
	@Qualifier("userDetailsService")
	private UserDetailsService userDetailsService;

	@Autowired
	private AuthenticationManager authenticationManager;

	@Value("${gigy.oauth.tokenTimeout:3600}")
	private int expiration;

	@Bean
	public PasswordEncoder passwordEncoder() {
		return new BCryptPasswordEncoder();
	}

	@Override
	public void configure(AuthorizationServerEndpointsConfigurer configurer) throws Exception {
		configurer.authenticationManager(authenticationManager);
		configurer.userDetailsService(userDetailsService);
	}

	@Override
	public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
		clients.inMemory().withClient("gigy").secret("secret").accessTokenValiditySeconds(expiration)
				.scopes("read", "write").authorizedGrantTypes("password", "refresh_token").resourceIds("resource");
	}
}
```

Let's stop here for a moment and take a look at what's going on. First of all, this is the first time we used `@Configuration` in our project. This annotation indicates that the annotated class contains bean definitions that will be loaded into Spring context. `@EnableAuthorizationServer` annotation will, similarly to `@EnableResourceServer`, let us use this application as an auth server.

Autowiring the `UserDetailsService`, that we added earlier, will enable us to use the users from our database in our auth server. `AuthenticationManager` is a Spring bean for handling the authenticated requests. We will not make any modifications to it in this part and will simply pass it to the `configure` method.

Notice that we have two methods with this name that are used for different purposes. The first one is used to hook up the users into the auth server (these come from our database  - or from our in-memory mock data we created earlier) and the second one configures the clients (applications connecting to the server). In this case, there is only one in-memory client called 'gigy'. Instead of hard-coding the clients in our configuration, we could use JDBC store, instead, but we are going to make it very simple in this tutorial.

This is everything we need to secure our server with OAuth2. Let's give it a try!

Let's try accessing some protected resource without authentication first:

```
curl -i -H "Accept: application/json" -X GET http://localhost:8000/gigy/people
```

This should result in an `401` error for unauthorized request. We need to get the auth token before sending requests to the protected endpoints. As we mentioned earlier, we are using the `password` grant type which means we are getting the token on behalf of a specific user so we need to send the user's credentials to the auth server. In this case, our sample application is running on localhost but in production. This endpoint should be always contacted over HTTPS.

```
curl -X POST --user 'gigy:secret' -d 'grant_type=password&username=peter@example.com&password=password' http://localhost:8000/gigy/oauth/token
```

The following response with the access and refresh tokens will be produced:

```json
{"access_token":"27c1d964-fcad-470f-b32b-219c662e6099","token_type":"bearer","refresh_token":"d7fe669c-cf46-46ee-b790-a9ef39ea7e63","expires_in":3599,"scope":"read write"}
```

We can now try to access the protected endpoint with our access token in the request headers:

```
curl -i -H "Accept: application/json" -H "Authorization: Bearer $TOKEN" -X GET http://localhost:8000/gigy/people
```

## Getting security context
There are multiple ways how to retrieve the current security context in a Spring/Spring Boot application. A nice summary can be found, for example, [here](http://www.baeldung.com/get-user-in-spring-security).

Since the `Person` entity represents our users, we first need to link it to the `User` object which is used by Spring security context. There are many ways to do that but we are going to simply use the username to associate these entities. It might not be the most elegant approach but it will do for our simple scenario. Let's modify our database schema:

```sql
CREATE TABLE people (
    person_id BIGINT PRIMARY KEY auto_increment,
    name VARCHAR(32),
    username VARCHAR(128) UNIQUE REFERENCES users (username),
    age INT,
);
```

And don't forget to modify the `Person` entity as well: `private String username;`. Finally, let us load people by their username:

```
@Repository
public interface PersonRepository extends CrudRepository<Person, Long> {
	Collection<Person> findAll();
	Person findByUsername(String username);
}
```

We are going to demonstrate how to access the currently logged-in user inside a controller. We already have an endpoint for deleting users in our application. However, the way it is implemented now allows anyone to delete any user. Let's say that the person entity should be deleted only by it's owner. This can be easily implemented by accessing the `Principal` object inside our controller:

```java
@RequestMapping(value = "/{id}", method = RequestMethod.DELETE)
public ResponseEntity<Void> deletePerson(@PathVariable long id, Principal principal) {
	Person currentPerson = personRepo.findByUsername(principal.getName());
	
	if (currentPerson.getId() == id) {
		personRepo.delete(id);
		return new ResponseEntity<Void>(HttpStatus.OK);
	} else {
		return new ResponseEntity<Void>(HttpStatus.UNAUTHORIZED);
	}
}
```

Now, the logged-in user cannot delete any entities (people) that he does not own. We would probably need to update the other endpoints as well to make them work with the 'current user' functionality but I will leave that for you as an exercise :-)

## Summary
Today, we learned how to protect our REST API using Spring Security and OAuth2. As always, you can download the working example from [Git](https://github.com/gigsterous/gigy-example/releases/tag/v4).
