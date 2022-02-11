---
layout: post
title: Spring Security -- 5) Filter Chain, Custom filter and Authentication
date: 2020-12-30 23:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

Let's look at the Filter Chain, more specifically `AuthenticationFilter` in the Spring Security. And also I am going to implement custom filter. This custom filter will include my custom authentication method because I do not want to use BasicAuthentication anymore!!

At the end of this post, you hopefully can be able to implement any custom authentication logic from top to the bottom using Spring Security.

Topics are:

- [**Github Link**](#github_link)
- [**Default Project Setup**](#default_project_setup)
- [**Overall architecture in one Picture**](#overall_architecture)
- [**Filter Chains in Spring**](#filter_chains_in_spring)
- [**What my Custom Authentication Method does**](#my_custom_auth_method)
- [**Creating the custom filter**](#custom_filter)
  - [Create a custom filter](#create_custom_filter)
  - [Creating the AuthenticationManager](#create_auth_manager)
  - [Creating the Authentication instance](#create_auth_instance)
  - [Creating the AuthenticationProvider](#create_auth_provider)
    - Add the custom provider to the configuration
  - [Add custom filter to the configuration](#add_filter_to_configuration)
- [**Run the application**](#starting_project)
- [**Modify the error response**](#modify_error_response)
- [**OncePerRequestFilter**](#once_per_requests_filter)

## Github Link <a name="github_link"></a>

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/spring-security-examples/tree/master/spring-security-custom-filter)

## Default Project Setup <a name="default_project_setup"></a>

You only need two dependencies: web and spring-security

```xml
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

And one rest endpoint:

```java
package com.mehmetozanguven.springsecuritycustomfilter.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @GetMapping("/hello")
    public String hello(){
        return "hello";
    }
}
```

## Overall architecture in one Picture <a name="overall_architecture"></a>

<img src="/assets/spring/security/custom_filter/custom_filter.png" alt="custom_filter.png" />

## Filter Chains in Spring <a name="filter_chains_in_spring"></a>

First thing first, there isn't only one filter called **AuthenticationFilter**. Instead there are many filters where chain pattern is applied. Each chain executes its responsibilities and move forward to the next chain.

> To learn more about the **chain of responsibility** pattern, you can refer to this [link](https://refactoring.guru/design-patterns/chain-of-responsibility)

I can visualize chain process in this picture, as you can see some of the filters are responsible for only logging, and some of them are responsible for authentication etc..

You can also add your custom filter **before and after specific chain**

<img src="/assets/spring/security/custom_filter/spring_filters.png" alt="spring_filters.png" />

## What my Custom Authentication Method does <a name="my_custom_auth_method"></a>

In my custom filter, if request contains a header called **CustomAuth** and value of that parameter is **password**, then I will say that "authenticate this request(s) and let request to be access the endpoint otherwise not"

## Creating the custom filter <a name="custom_filter"></a>

You should follow the same structure as spring follows when creating a custom filter. What I mean, you should create filter(s), authentication manager and also you should create provider(s) for that filter(s). Provider(s) that you are going to implement, will contain the custom Authentication Logic.

Then let's summarize the flow in spring.

If you could use Basic Authentication, flow would be:

- BasicAuthenticationFilter(will pass the request to the Manager) -> AuthenticationManager(will find the correct provider and pass the request to the provider) -> AuthenticationProvider(calls the UserDetailsService#loadUserByUsername). After getting user's details from the UserDetailsService, AuthenticationProvider will check the password via correspond PasswordEncoder.

If you could implement your own Filter, flow would be the same with Basic Authentication but this time you will use your own provider,manager etc..

### Create a custom filter <a name="create_custom_filter"></a>

You can implement the generic java servlet `Filter` class. After implementing Filter class, you must write your logic in `doFilter(..)` method.

> Generating custom filter via implementing the Filter class is more generic. In this case you are dealing with ServletRequest which is the generic servlet component. That would be better to deal with kind of a class that also supports HTTP things. As you can guess, there is a specific servlet for http specific things called `HttpServletRequet`. To work with you can extend you custom filter with `org.springframework.web.filter.OncePerRequestFilter` instead of `Filter` interface.
>
> Because I will look at the request's header to implement authentication logic and because header is the specific thing in the HTTP, I must convert ServletRequest to the HttpServletRequest.
>
> Or I could directly use **OncePerRequetFilter**
>
> Keep reading ...

In the `doFilter(..)` method:

- It will be called by the servlet container for each time when request/response pair is passed through the chain.
- You can do some setup/update the request before other chains process. `FilterChain` will forward the request and response to the other chains. (If you read the **chain of responsibility pattern in [here](https://refactoring.guru/design-patterns/chain-of-responsibility), you would be more comfortable**)
- You can do some setup/update before the response was sent to the client

```java
package com.mehmetozanguven.springsecuritycustomfilter.security.filters;

import javax.servlet.*; // make sure you import javax.servlet
import java.io.IOException;

@Component
public class MyCustomAuthenticationFilter implements Filter {

    // because I am using custom filter then there must be a manager for that,
    // but I didn't define any manager YET ??
    @Autowired
    private AuthenticationManager authenticationManager;

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {

    }
}
```

Now filter class is done, but there is no AuthenticationManager, I should define the one. I can define the manager in my configuration class extending with `WebSecurityConfigurerAdapter`.

### Creating the AuthenticationManager <a name="create_auth_manager"></a>

To create Manager, I will just override one method inside the `WebSecurityConfigurerAdapter`

> Override `authenticationManagerBean()` to expose the AuthenticationManager from the WebSecurityConfigurerAdapter

```java
package com.mehmetozanguven.springsecuritycustomfilter.config;

import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@Configuration
public class ProjectBeanConfiguration extends WebSecurityConfigurerAdapter {

    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
}

```

```java
public interface AuthenticationManager {
	Authentication authenticate(Authentication authentication) throws AuthenticationException;
}
```

**AuthenticationManager** will run the `authenticate(Authentication authentication)` method, then pass the Authentication object to the `AuthenticationProvider` Then as you recall from the previous [blog](https://mehmetozanguven.github.io/spring/2020/12/30/spring-security-4-authentication-provider.html#custom_authentication_provider), options you have when running the `authenticate(authentication)` method:

1. `authenticate(authentication)` should return fully `Authentication` object.
2. If authentication fails, should throw an authentication exception

But first, You should create an **Authentication** instance

### Creating the Authentication instance <a name="create_auth_instance"></a>

In this post, I am going to use the `UsernamePasswordAuthenticationToken` which implements the `Authentication`.

```java
package com.mehmetozanguven.springsecuritycustomfilter.security.authentication;

import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.GrantedAuthority;

import java.util.Collection;

public class MyCustomAuthentication extends UsernamePasswordAuthenticationToken {
	// this constructor creates a Authentication instance which is not fully authenticated
    public MyCustomAuthentication(Object principal, Object credentials) {
        super(principal, credentials);
    }

    // this constructor creates a Authentication instance which is fully authenticated
    public MyCustomAuthentication(Object principal, Object credentials, Collection<? extends GrantedAuthority> authorities) {
        super(principal, credentials, authorities);
    }
}
```

Now let's update the filter class:

```java
public class MyCustomAuthenticationFilter implements Filter {
    private static final Logger logger = LoggerFactory.getLogger(MyCustomAuthenticationFilter.class.getSimpleName());

    private AuthenticationManager authenticationManager;

    public MyCustomAuthenticationFilter(AuthenticationManager authenticationManager) {
        this.authenticationManager = authenticationManager;
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest httpServletRequest = (HttpServletRequest) servletRequest;
        String authorization = httpServletRequest.getHeader("CustomAuth");

        MyCustomAuthentication customAuthentication = new MyCustomAuthentication(authorization, null); // (Object principal, Object credentials)
        Authentication authResult = authenticationManager.authenticate(customAuthentication);

        // In real case, there is no need to check isAuthenticated method
        // because when authentication fails, it must throw an error
        if (authResult.isAuthenticated()){
            // If I have fully authentication instance, add to the security context
            // do not think about the security context for now..
            SecurityContextHolder.getContext().setAuthentication(authResult);

            filterChain.doFilter(servletRequest, servletResponse);
        }
    }
}
```

`authenticationManager.authenticate(customAuthentication);` will call the AuthenticationProvider. But I didn't defined one YET. Let's define the custom authentication provider.

### Creating the AuthenticationProvider <a name="create_auth_provider"></a>

As you recall from the previous blog, you must determine when this provider must be applied. To do that you must override the `supports` method:

```java
package com.mehmetozanguven.springsecuritycustomfilter.security.providers;

public class MyCustomAuthenticationProvider implements AuthenticationProvider {

	//...

   /**
     * If authentication type is MyCustomAuthentication, then
     *  apply this provider
     * @param authentication
     * @return
     */
    @Override
    public boolean supports(Class<?> authentication) {
        return MyCustomAuthentication.class.equals(authentication);
    }
}
```

The left is to define the authentication logic:

```java
public class MyCustomAuthenticationProvider implements AuthenticationProvider {
    private final String secretKey = "password";

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String requestKey = authentication.getName();
        if (requestKey.equals(secretKey)){
            MyCustomAuthentication fullyAuthenticated = new MyCustomAuthentication(null, null, null);
            return fullyAuthenticated;
        }else{
            throw new BadCredentialsException("Header value is not correct");
        }
    }
    // ...
}
```

Add this custom provider to the configuration:

```java
@Configuration
public class ProjectBeanConfiguration extends WebSecurityConfigurerAdapter {

    public MyCustomAuthenticationProvider myCustomAuthenticationProvider() {
        return new MyCustomAuthenticationProvider();
    }

    /**
     * Configure the custom provider
     * @param auth
     * @throws Exception
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.authenticationProvider(myCustomAuthenticationProvider());
    }

    @Override
    @Bean
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
}
```

Typically, you would have password encoder and you would check the password provided via `authentication.getCredentials()` and the one you would get from the UserDetailsService, but right now I am implementing my custom authentication logic and I do not need any passwordEncoder. If request's header named **CustomAuth** value is equal to **password**, then it is okey for me.

### Add custom filter to the configuration <a name="add_filter_to_configuration"></a>

At the beginning of the post, I have decided to use my filter instead of the BasicAuthenticationFilter, that's why I have used `http.addFilterAt(..)` which replace the filter with my custom one.

```java
@Configuration
public class ProjectBeanConfiguration extends WebSecurityConfigurerAdapter {

    public MyCustomAuthenticationFilter myCustomAuthenticationFilter() throws Exception {
        return new MyCustomAuthenticationFilter(authenticationManagerBean());
    }

	// ...

    /**
     * Configure the filter, basically replace the
     *  BasicAuthenticationFilter with the custom one
     * @param http
     * @throws Exception
     */
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.addFilterAt(myCustomAuthenticationFilter(), BasicAuthenticationFilter.class);
    }
}
```

## Run the application <a name="starting_project"></a>

After starting the application, run the following curl command:

```bash
[mehmetozanguven@localhost mehmetozanguven.github.io]$ curl -H "CustomAuth:password" -X GET http://localhost:8080/hello
hello
```

To imitate the failing:

```bash
[mehmetozanguven@localhost mehmetozanguven.github.io]$ curl -H "CustomAuth:passwordX" -X GET http://localhost:8080/hello

<!doctype html><html lang="en"><head><title>HTTP Status 500 – Internal Server Error</title><style type="text/css">body {font-family:Tahoma,Arial,sans-serif;} h1, h2, h3, b {color:white;background-color:#525D76;} h1 {font-size:22px;} h2 {font-size:16px;} h3 {font-size:14px;} p {font-size:12px;} a {color:black;} .line {height:1px;background-color:#525D76;border:none;}</style></head><body><h1>HTTP Status 500 – Internal Server Error</h1></body></html>
```

After failing look at the console:

```wiki
2020-12-30 23:03:13.134 ERROR 20575 --- [nio-8080-exec-7] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception

org.springframework.security.authentication.BadCredentialsException: Header value is not correct
	at com.mehmetozanguven.springsecuritycustomfilter.security.providers.MyCustomAuthenticationProvider.authenticate(MyCustomAuthenticationProvider.java:21) ~[classes/:na]
```

As you can see, response seems ugly, let's modify it .

## Modify the response <a name="modify_error_response"></a>

```java
@Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest httpServletRequest = (HttpServletRequest) servletRequest;
        HttpServletResponse httpServletResponse = (HttpServletResponse) servletResponse;

        String authorization = httpServletRequest.getHeader("CustomAuth");

        MyCustomAuthentication customAuthentication = new MyCustomAuthentication(authorization, null);

        try{
            Authentication authResult = authenticationManager.authenticate(customAuthentication);

            // In real case, there is no need to check isAuthenticated method
            // because when authentication fails, it must throw an error
            if (authResult.isAuthenticated()){
                // If I have fully authentication instance, add to the security context
                // do not think about the security context for now..
                SecurityContextHolder.getContext().setAuthentication(authResult);
                filterChain.doFilter(servletRequest, servletResponse);
            }else{
                httpServletResponse.setStatus(HttpServletResponse.SC_FORBIDDEN);
            }
        } catch (AuthenticationException authenticationException) {
            httpServletResponse.setStatus(HttpServletResponse.SC_FORBIDDEN);
        }
    }
```

## OncePerRequestFilters <a name="once_per_requests_filter"></a>

As I said previously, instead of wrapping the ServletRequest with HttpServletRequest, you can use it directly via extending `OncePerRequestFilters`

```java
public class MyCustomAuthenticationFilter extends OncePerRequestFilter {

    private AuthenticationManager authenticationManager;

    public MyCustomAuthenticationFilter(AuthenticationManager authenticationManager) {
        this.authenticationManager = authenticationManager;
    }

    @Override
    public void doFilterInternal(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, FilterChain filterChain) throws IOException, ServletException {

        String authorization = httpServletRequest.getHeader("CustomAuth");

        MyCustomAuthentication customAuthentication = new MyCustomAuthentication(authorization, null);

        try{
            Authentication authResult = authenticationManager.authenticate(customAuthentication);

            // In real case, there is no need to check isAuthenticated method
            // because when authentication fails, it must throw an error
            if (authResult.isAuthenticated()){
                // If I have fully authentication instance, add to the security context
                // do not think about the security context for now..
                SecurityContextHolder.getContext().setAuthentication(authResult);
                filterChain.doFilter(httpServletRequest, httpServletResponse);
            }else{
                httpServletResponse.setStatus(HttpServletResponse.SC_FORBIDDEN);
            }
        }catch (AuthenticationException authenticationException){
            httpServletResponse.setStatus(HttpServletResponse.SC_FORBIDDEN);
        }
    }
}
```

I will continue with the next one ...
