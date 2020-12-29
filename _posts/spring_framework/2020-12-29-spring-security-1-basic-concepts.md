---
layout: post
title:  Spring Security -- 1) Basic Concepts "
date:   2020-12-29 12:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

In these short series , I am going to dive into what Spring Security is, how Spring Security works. Most of the example application would be for web environment. Because I am going to use spring boot you should also use it or you have to to some setup to work with xml setup and others. 

Topic are:

- [**Github Link**](#github_link)
- [**Default Project Setup**](#default_setup)
- [**What is the Spring Security?**](#what_is_spring_security)
- [**Secure the endpoints**](#secure_endpoints)
- [**Overriding the default configuration**](#default_configuration)
  - If you override the `UserDetailsService` you must also override the `PasswordEncoder`

## Github Link <a name="github_link"></a>

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/spring-security-examples/tree/master/spring-security-basic) 


## Default Project Setup <a name="default_setup"></a>

Before diving into please create a new spring boot application which includes only these dependencies:

```xml
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.3.5.RELEASE</version>
		<relativePath/>
	</parent>
	<groupId>com.mehmetozanguven</groupId>
	<artifactId>springsecurityexample</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>springsecurityexample</name>
	<description>Demo project for Spring Security using Spring Boot</description>

	<properties>
		<java.version>11</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.security</groupId>
			<artifactId>spring-security-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
```



## What is the Spring Security? <a name="what_is_spring_security"></a>

Spring Security is a powerful and highly customizable authentication and access-control framework. It is the de-facto standard for securing  Spring-based applications. It is a framework that focuses on providing both authentication and authorization to Java applications. 

<img src="/assets/spring/security/basic_concepts/spring_security_basic.png" alt="spring_security_basic.png" />

- When request is intercepted, `AuthenticationFilter` pass your credentials (username:password) to the ` AuthenticationManager`. Therefore responsibility of AuthenticationFilter is just pass the credentials to the AuthenticationManager
- AuthenticationManager delivers to the responsibility of the authentication to the one of the `AuthenticationProvider` s. (AuthenticationManager will find the proper AuthenticationProvider for the request(s))
- In the AuthenticationProvider, we will have the Authentication Logic.
- Most probably requests will include username and password, therefore you should identify these in your database, cache system etc.. We need find out the user with credentials somehow, in spring security component that finds out the correct user is called `UserDetailsService` (UserDetailsService responsibility is the to find out the user in the database via credentials in the requests)
- The last component that acts on the authentication is the `PasswordEncoder` which is used by the AuthenticationProvider to implement authentication logic. Role of PasswordEncoder is to check whether password is correct or not. After UserDetailsService gets the details of the user from the database and these details also contain user's password, then PasswordEncoder will try to match the passwords in the request(s) and the user's details from the database.
- Now consider that we have valid authentication, if that is the case, then valid authentication response will forward the AuthenticationFilter,  in the AuthenticationFilter, spring will store the authenticated object in the `SecurityContext` 
- After all we can get the authenticated user object using the SecurityContext.

That is the main architecture of Spring Security.

## Secure the endpoints <a name="secure_endpoints"></a>

Now let's create a controller and secure it.

```java
package com.mehmetozanguven.springsecurityexample.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class SampleController {

    @GetMapping("/test")
    public String test(){
        return "test";
    }
}
```

If you run the application, because I am using the spring boot, spring-boot-starter-security dependency will protect the endpoint `/test` , the reason is that this dependency called `spring-boot-starter-security` added some default configuration for our spring application. Spring security will create a static username  (which is **user**) and generated password in the console:

```wiki
2020-12-27 16:41:07.760  INFO 18000 --- [           main] .s.s.UserDetailsServiceAutoConfiguration : 

Using generated security password: 539983a1-3724-4fde-b757-da3818bc8f16
...
```

If you call this endpoint via curl (or postman), you will not access it:

```bash
[mehmetozanguven@localhost ~]$ curl -X GET http://localhost:8080/test | jq .
{
  "timestamp": "2020-12-27T13:42:38.169+00:00",
  "status": 401,
  "error": "Unauthorized",
  "message": "",
  "path": "/test"
}
```

Because the default spring security configuration is the Basic Authentication, just add the basic authentication in your curl command: 

> Removed `.jq` pipeline because i am not returning json from the endpoint. jq is the command line JSON processor

```bash
[mehmetozanguven@localhost ~]$ curl --user user:539983a1-3724-4fde-b757-da3818bc8f16 -X GET  http://localhost:8080/test
test
```

All these setups is done by `BasicAuthenticationFilter`:

```java
package org.springframework.security.web.authentication.www;

public class BasicAuthenticationFilter extends OncePerRequestFilter {
	@Override
	protected void doFilterInternal(HttpServletRequest request,
			HttpServletResponse response, FilterChain chain)
					throws IOException, ServletException {
			// ....
			if (authenticationIsRequired(username)) {
				Authentication authResult = this.authenticationManager
						.authenticate(authRequest);
				// That is the last step, writing the authenticated object to the SecurityContext
                // to get back authenticated object for future logics..
				SecurityContextHolder.getContext().setAuthentication(authResult);
			}
			// ....
	}
}
```

Now let's override the default configuration for spring security

## Overriding the configuration <a name="default_configuration"></a>

Let's create our own user instead of the generated one from spring security, right now I will store the user in the memory. 

In my scenario I am just overriding the `UserDetailsService`  and do not forget that overriding the UserDetailsService forces us to override the PasswordEncoder

In default configuration, because there is no UserDetailsService and PasswordEncoder, these ones are generated by spring security.

There are multiple ways to configure spring security, I will start with basic one: creating the bean of type UserDetailsService:

```java
package com.mehmetozanguven.springsecurityexample.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;

@Configuration
public class ProjectBeanConfiguration {

    @Bean
    public UserDetailsService userDetailsService(){
        InMemoryUserDetailsManager inMemoryUserDetailsManager = new InMemoryUserDetailsManager();
        UserDetails testUser = User.withUsername("testUser")
                    .password("1234")
                    .authorities("read")
                    .build();
        inMemoryUserDetailsManager.createUser(testUser);
        return inMemoryUserDetailsManager;
    }
}
```

In the configuration `InMemoryUserDetailsManager` implements `UserDetailsManager` interface which extends ` UserDetailsService` 

After that I am just creating the testUser type of UserDetails (that is the type spring security requests), and testUser has a username **testUser** with password: **1234** and one authority: **read** (testUser can read something)

After all I am just adding the testUser to the my InMemoryUserDetailsManager. At the end I just defined the my own UserDetailsService

Now, if I run the project, I will not see **Using generated security password: ...**  in the console, because I just told spring to use my own userDetailsService. However, because there is no default configuration for UserDetailsService there won't be also default PasswordEncoder and if you run this curl command, you will get an exception:

```bash
[mehmetozanguven@localhost ~]$ curl --user testUser:1234 -X GET  http://localhost:8080/test
[mehmetozanguven@localhost ~]$ 
```

and the exception will be:

```wiki
java.lang.IllegalArgumentException: There is no PasswordEncoder mapped for the id "null"
	at org.springframework.security.crypto.password.DelegatingPasswordEncoder$UnmappedIdPasswordEncoder.matches(DelegatingPasswordEncoder.java:250) ~[spring-security-core-5.3.5.RELEASE.jar:5.3.5.RELEASE]
...
```

As I mentioned previously **if you create your own UserDetailsService, you must create also PasswordEncoder**, because there is no PasswordEncoder, authentication process fails.

Let's create a passwordEncoder

### Creating PasswordEncoder

```java
@Configuration
public class ProjectBeanConfiguration {

    @Bean
    public UserDetailsService userDetailsService(){
        // ...
    }
    
    @Bean
    public PasswordEncoder passwordEncoder(){
        return NoOpPasswordEncoder.getInstance();
    }
}
```

> Don't use NoOpPasswordEncoder in production,  NoOpPasswordEncoder is provided for legacy and testing purposes only and is not considered secure.

Now let's run the application again:

````bash
[mehmetozanguven@localhost ~]$ curl --user testUser:1234 -X GET  http://localhost:8080/test
test
````

Now correct user can access the authenticated endpoints..

That's the basic security in Spring Security, I will continue with the next post. 