---
layout: post
title: Spring Security -- 7) Security Context and Security Context Holder
date: 2021-05-11 22:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

In this post, let's find out **what the Security Context is**

> I am going to use the project that I have implemented in the previous [post](https://mehmetozanguven.github.io/spring/2021/01/06/spring-security-6-multiple-filters-and-providers.html).
>
> Here is the github [link](https://github.com/mehmetozanguven/spring-security-examples/tree/master/spring-security-multiple-providers) for previous project

Topics are:

- [**Github Link**](#github_link)
- [**Recap the previous project**](#recap_previous_project)
  - **Where To Store Authenticated User**
- [**Security Context**](#security_context)
  - **How to obtain Authenticated User**
    - **Endpoint Level (Controller Level)**
    - **Using SecurityContextHolder**
- [**What happens for more than one thread (Reactive applications etc..)**](#more_than_one_thread)
- [**SecurityContextHolder**](#securityContextHolder)
  - **How to change the Strategy**
  - **MODE_INHERITABLETHREADLOCAL**
  - **Access SecurityContext Without Changing the Strategy**

## Github Link <a name="github_link"></a>

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/spring-security-examples/tree/master/spring-security-context)

## Recap the previous project <a name="recap_previous_project"></a>

> My OTP application can not be considered as secure. There are two main reasons:
>
> 1. OTP codes does not have any expiration time
> 2. Authorization header is stored in the hash map not in the database, could (most probably) lead to the memory leak

In the previous post, I have implemented OTP(one time password) with spring boot. Basically what I am doing:

- Users try to access to the system via his/her username and password
- If username/password is correct, then Spring boot application generates an OTP and store it to the database. (Generally generated OTP will send to the user via mobile phone/email address etc..)
- After user gets the OTP, right now he/she tries to access with otp (means that username is the username from the previous setup, but there will be no password header, he/she will send the otp with header name otp)
- If the second request is correct (otp and username), then Spring boot application returns Authorization token to allow user to access restricted endpoints

Let's try it: (I have already defined an user in the database --username: test_user, password: 1234 --)

- First, try to login with wrong credentials:

```bash
curl -H "username:test_user" -H "password:1234x" -X GET http://localhost:8080/login
```

```json
{
  "timestamp": "2021-05-11T19:04:06.171+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "message": "",
  "path": "/login"
}
```

- Try to login with correct credentials:

```bash
curl -H "username:test_user" -H "password:1234" -X GET http://localhost:8080/login
```

- Look at the table **otp**

```sql
testdatabase=# select * from otp;
 id | username  |    otp
----+-----------+------------
  1 | test_user | VxxLuQMzzG
-- assume that this otp will send to the user via mobile phone
```

- Now try to get access token via otp

```bash
curl -H "username:test_user" -H "otp:VxxLuQMzzG" -v -X GET http://localhost:8080/login

Note: Unnecessary use of -X or --request, GET is already inferred.
*   Trying ::1:8080...
* Connected to localhost (::1) port 8080 (#0)
> GET /login HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.69.1
> Accept: */*
> username:test_user
> otp:VxxLuQMzzG
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200
< Authorization: 9a98d974-6375-40cc-9086-6d93f955a1ee # HERE IS THE OUR TOKEN
< X-Content-Type-Options: nosniff
< X-XSS-Protection: 1; mode=block
< Cache-Control: no-cache, no-store, max-age=0, must-revalidate
< Pragma: no-cache
< Expires: 0
< X-Frame-Options: DENY
< Content-Length: 0
< Date: Tue, 11 May 2021 19:07:34 GMT
<
* Connection #0 to host localhost left intact
```

Now we can access to the restricted endpoints:

```bash
curl -H "Authorization:9a98d974-6375-40cc-9086-6d93f955a1ee" -X GET http://localhost:8080/hello

hello
```

### Where To Store Authenticated User

In the previous project, we set the authenticated user in the filter, after getting the fully authenticated instance.

```java
public class TokenAuthFilter extends OncePerRequestFilter {
	// ...
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        String authorization = request.getHeader("Authorization");
		// create not-fully authenticated instance
        TokenAuthentication tokenAuthentication = new TokenAuthentication(null, authorization);
		// authenticationManager will find the correct provider
        // and provider will return either fully authentication instance or
        // throw an error
        Authentication authResult = authenticationManager.authenticate(tokenAuthentication);
        // Set the authentication in the SecurityContext
        SecurityContextHolder.getContext().setAuthentication(authResult);
        filterChain.doFilter(request, response);
    }
}
```

If everything is okey, let's discuss the SecurityContext

## Security Context <a name="security_context"></a>

First, you can access SecurityContext in the anywhere in the application

### How to obtain Authenticated User

#### Endpoint Level (Controller Level)

In the endpoint level, easiest way to get authenticated user is to add `Authentication` parameter to the method.

> If you look at the my previous post, when we are generating fully authenticated request via returning to the instance type of `Authentication`

```java
package com.mehmetozanguven.springsecuritymultipleproviders.controllers;

import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello(Authentication authentication){
        return "hello: " + authentication.getName();
    }
}
```

```bash
curl -H "Authorization:74bc9635-8b9c-49f6-83d2-671f0ff7c20f" -X GET http://localhost:8080/hello
hello: 74bc9635-8b9c-49f6-83d2-671f0ff7c20f
```

`authentication.getName()` returns the token because in the `TokenAuthProvider` we set the principle as authorization token:

```java
@Component
public class TokenAuthProvider implements AuthenticationProvider {
    // ...
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
      	// ...
        if (isCorrectToken){
            return new TokenAuthentication(authorizationToken, authorizationToken, List.of(() -> "read"));
        } else {
            throw new BadCredentialsException("Authorization value is not correct");
        }
    }
	// ...
}
```

#### Using SecurityContextHolder

We can use the `SecurityContextHolder` class:

```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello(){
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        return "hello: " + authentication.getName();
    }
}
```

## What happens for more than one thread <a name="more_than_one_thread"></a>

In traditional way, when we do a request, application generates one thread. And this thread can handle the entire request. Problem arises when we have separate thread, how can we access the authenticated object in another thread ?

Just update the controller with `@Async`. This will generate another thread. (Also enable the async in the configuration)

```java
@Configuration
@EnableAsync
public class ProjectBeanConfiguration extends WebSecurityConfigurerAdapter {
    //...
}
```

```java
@RestController
public class HelloController {

    @GetMapping("/hello")
    @Async
    public String hello(){
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        return "hello: " + authentication.getName();
    }
}
```

Run the application, and send request to the endpoint

```bash
curl -H "Authorization:f95742dd-9733-43aa-9776-8eaa95fb3130" -X GET http://localhost:8080/hello
```

Even if you get the 200 OK, look at the console, you will see the `NullPointerException` , because Authentication is null.

```java
java.lang.NullPointerException: null
	at com.mehmetozanguven.springsecuritymultipleproviders.controllers.HelloController.hello(HelloController.java:16) ~[classes/:na]
	at com.mehmetozanguven.springsecuritymultipleproviders.controllers.HelloController$$FastClassBySpringCGLIB$$c071c780.invoke(<generated>) ~[classes/:na]
```

This happens because default thread (per request thread) will return 200 and actually this thread contains the authenticated user inside `SecurityContextHolder.getContext().getAuthentication()` . However, using `@Async` annotation, Spring will run the method in the different thread, and that thread don't know the value of the `SecurityContextHolder.getContext().getAuthentication()` and it simply returns null.

Reason is that default strategy for the `SecurityContextHolder` is the **MODE_THREADLOCAL**

## **SecurityContextHolder** (<a name="securityContextHolder"></a> )

- It is just a class to manage `SecurityContext`
- `SecurityContextHolder` implements 3 strategies to manager `SecurityContext`
  - Default one **MODE_THREADLOCAL** (Store information and make it accessible for the specific thread, only thread_1 stores a value on the thread local, thread_2 can not access it, only available for thread_1)
  - **MODE_INHERITABLETHREADLOCAL** (Copy the securityObject from the parent thread to the child thread)

### How to change the Strategy

We can change the strategy when spring initializing the beans in the configuration file:

```java
@Configuration
@EnableAsync
public class ProjectBeanConfiguration extends WebSecurityConfigurerAdapter {
    // ...
    @Bean
    public InitializingBean initializingBean() {
        return () -> {
   	SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);
        };
    }
}
```

### MODE_INHERITABLETHREADLOCAL

Run the application after setting the strategy to:

```java
@Configuration
@EnableAsync
public class ProjectBeanConfiguration extends WebSecurityConfigurerAdapter {
    // ...
    @Bean
    public InitializingBean initializingBean() {
        return () -> {
   	SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);
        };
    }
}
```

Now, when you hit the `/hello` request, you will not get the exception. (Everything works fine)

```bash
curl -H "Authorization:8a16be9e-573f-45bf-a8f1-6059518b60c8" -X GET http://localhost:8080/hello
```

### Access SecurityContext Without Changing the Strategy

Sometimes you may need to access the security context without changing the strategy, in that case you can wrap your runnable with `DelegatingSecurityContextRunnable` .

> You can also wrap your `callable, executorService` with `DelegatingSecurityContextCallable, DelegatingSecurityContextExecutorService`

Now undo the `@Async` operations (remove `@EnableAsync` and `@Asycn`) also remove the `initializingBean()` in the configuration and update the hello method:

```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello(){
        Runnable runnable = () -> {
            Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
            System.out.println(authentication.getName());
        };
        DelegatingSecurityContextRunnable ds = new DelegatingSecurityContextRunnable(runnable);

        Thread thread = new Thread(ds);
        thread.start();
        return "hello";
    }
}
```

And send the request:

```bash
curl -H "Authorization:b6347da7-9a69-4f25-b672-3d44fe0b28be" -X GET http://localhost:8080/hello
hello
```

I will continue with the next one ...
