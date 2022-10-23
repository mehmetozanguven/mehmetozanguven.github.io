---
layout: post
title: Spring Security -- 9) Spring Security CORS Setup
date: 2021-05-21 22:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

In this post, let's find out **what is the CORS policy, how to implement in the Spring Boot and Spring Security, finally how to resolve most common error Access-Control-Allow-Origin => Missing**

Topics are:

- [**Github Link**](#github_link)
- [**What is the CORS?**](#what_cors_is)
  - [**What do we mean by different origin ?**](#what_different_origin_means)
  - [**Why do we need a Cors policy ?**](#why_we_need_cors_policy)
  - [**How CORS works ?**](#how_cors_works)
  - [**What do we mean by Simple Requests ?**](#what_simple_request_is)
- [**Project Setup**](#project_setup)
- [**Getting Response From the Same Origin**](#response_from_same_origin)
- [**Error: Access-Control-Allow-Origin => Missing**](#access_control_allow_origin_missing)
- [**CORS doesn't do anything about the call of the endpoint**](#cors_and_endpoint)
- [**How to resolve Access-Control-Allow-Origin => Missing Error**](#resolve_missing_error)
  - [**CORS support through Spring Security**](#cors_for_spring_security)
  - [**CORS Setup through MVC Application**](#cors_for_mvc)

## Github Link <a name="github_link" />

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/spring-security-examples/tree/master/spring-security-cors-setup)

## What is the CORS? <a name="what_cors_is" />

**Cross-Origin Resource Sharing (CORS) is a protocol that enables scripts running on a browser client to interact with resources from a different origin.**

> We need this policy, because `XMLHttpRequest` and `fetch` follows the **same-origin-policy** and that leads JavaScript can only make calls to URLs that lives on the same origin where the script is running.

First let's define what do we mean by different origin?

### What do we mean by different origin? <a name="what_different_origin_means" />

Two origins are different if they have:

- different schemes (`HTTP` OR `HTTPS`)
- different domains (`sample.com` VS `api.sample.com` VS `another.com`)
- different ports (`sample.com:8081` VS `sample.com:8080`)

### Why do we need a Cors policy ? <a name="why_we_need_cors_policy" />

In generally speaking, your web application should never interact with resources from a different origin. However today web application structure, you will mostly have an backend and frontend which are running on different ports. Therefore you somehow guarantee the communication between two sides.

> Web browsers can use headers related to CORS to determine whether or not an `fetch` call should continue or fail

There are a few headers, but most important one is the `Access-Control-Allow-Origin` **which tells browsers to allow that origin to access the resource**

- Example:
  - `Access-Control-Allow-Origin: *` => if your back-end application runs on the domain called `api.sample.com` than this header says that every other origin can access the `api.sample.com` resources. For instances, web browsers that run scripts on the domains `sample.com` or `another.com` can access(make request) to your domain: `api.sample.com`
  - `Access-Control-Allow-Origin: http://sample.com` => only the web browsers that run scripts on the domain `http://sample.com` can make request to your domain.

> Be carefully as a developer you are not responsible to make request, **web browser** will decide it.

### How CORS works ? <a name="how_cors_works" />

- Cors works by adding new **Http Headers** that let servers describe which origins are permitted to read that information from a web browser.

> Be careful, it says **... from a web browser** . That's means you can send a curl request to the server ? Right !!

- CORS specification says that any mutating request (requests that change something in the server such as `POST,PUT,DELETE` ) must be done in the following way:

  - First, browser must do a **preflight request** and ask the supported methods to the server with the **HTTP OPTIONS**. (Requests that do not need any preflight request are called **simple requests**)
  - After getting the approval from the server, then web browser can send the actual request.

  > Server can also inform the clients whether the credentials should be send with requests.

### What do we mean by Simple Requests ? <a name="what_simple_request_is" />

> For more detail explanation, please refer to the https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#simple_requests

A “simple request” is one that **meets all the following conditions**:

- One of the allowed methods:

  - **GET**
  - **HEAD**
  - **POST** (with limited content-types)

- Except for the headers automatically set by the user agent (such as Connection, User-Agent), the only headers, which are allowed to be manually set, are considered as simple request:

  - **Accept**
  - **Accept-Language**
  - **Content-Language**
  - **Content-Type** (but only allowed ones)

> In other words, even if you match with allowed methods, content-type, and other conditions, if there is an header except the ones above (and also except the ones that are automatically set by the user agent), then it will be not be considered as **simple request**

- The allowed values for Content-Type

  - `application/x-www-form-urlencoded`
  - `multipart/form-data`
  - `text/plain`

> Basically that's means if you send POST request with content-type `application/json`, web browsers will think that your request is not a simple request

- No `ReadableStream` object is used in the request

- If you send request with `XMLHttpRequest` and if you do any of the following one, then your request will not be considered as simple request:
  - If you register an event listener for the object returned by `XMLHttpRequest.upload`

You will understand better when we are working on spring boot application.

## Project Setup <a name="project_setup" />

To simulate CORS, first create a simple spring boot project. This project will contain the following dependencies:

- Spring Web
- Spring Security
- Thymeleaf

Create home page controller and corresponding html pages inside `resources/templates`

```java
@Controller
public class HomePageController {

    @GetMapping
    public String homePage() {
        return "homePage";
    }

    @PostMapping("/post")
    @ResponseBody
    public String ajaxRequest() {
        return "AJAX_RESPONSE_FROM_SPRING";
    }
}
```

- `homePage.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>HomePage</title>
  </head>
  <body>
    <h1>Home Page</h1>
    <p id="ajaxResponse"></p>

    <script>
      async function getData(url) {
        var response = await fetch(url, { method: "POST" });
        var result = await response.text();
        var ajaxResponse = document.getElementById("ajaxResponse");
        ajaxResponse.innerHTML = result;
      }
      const url = "http://localhost:8080/post";
      getData(url);
    </script>
  </body>
</html>
```

- For this sample project, I will disable the CSRF protection (don't do this in production)
- And also I will allow all request to be accessed without login.

```java
@Component
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable();
        http.authorizeRequests().anyRequest().permitAll();
    }
}
```

## Getting Response From the Same Origin <a name="response_from_same_origin" />

After run the project, you will see the **AJAX_RESPONSE_FROM_SPRING** in the page. Because you are in the same origin as the web application

## Error: Access-Control-Allow-Origin => Missing <a name="access_control_allow_origin_missing" />

Instead of opening the `http://localhost:8080/`, open the `http://127.0.0.1:8080/`. In the web console you will encounter the error like this one:

```wiki
Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at http://localhost:8080/post. (Reason: CORS header ‘Access-Control-Allow-Origin’ missing).
```

You will see this response, because you are sending request in different origin and `fetch API` blocks the response

Open the network tab when you are sending request from 127.0.0.1, and find your post request. Here is the my request's header:

```wiki
POST /post HTTP/1.1
Host: localhost:8080
User-Agent: Mozilla/5.0 (X11; Fedora; Linux x86_64; rv:88.0) Gecko/20100101 Firefox/88.0
Accept: */*
Accept-Language: en-US,tr;q=0.7,en;q=0.3
Accept-Encoding: gzip, deflate
Referer: http://127.0.0.1:8080/
Origin: http://127.0.0.1:8080
Connection: keep-alive
Pragma: no-cache
Cache-Control: no-cache
Content-Length: 0
```

Please look at the origin field. This field says that request was generated from the origin `http://127.0.0.1:8080`
Now look at the response's headers:

```wiki
HTTP/1.1 200
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
X-Frame-Options: DENY
Content-Type: text/plain;charset=UTF-8
Content-Length: 25
Date: Thu, 13 May 2021 10:37:44 GMT
Keep-Alive: timeout=60
Connection: keep-alive
```

As you can see, Spring did not return any header(s) related to the CORS (such `Access-Control-Allow-Origin`), which Spring basically tells the browsers:

- Hey browsers when a script makes a request, then show the response, if and only if that request was made from the origin `http://localhost:8080`, otherwise blocks the response.

> **Be careful it blocks the response, that doesn't mean browser also blocks the request(s)**.
>
> In other words, **cors doesn't do anything about the call of the endpoint**

## CORS doesn't do anything about the call of the endpoint <a name="cors_and_endpoint" />

Update the post method and send the ajax request from the origin `http:127.0.0.1` :

```java
package com.mehmetozanguven.springsecuritycorssetup.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
public class HomePageController {
	// ..

    @PostMapping("/post")
    @ResponseBody
    public String ajaxRequest() {
        System.out.println("Method called");
        return "AJAX_RESPONSE_FROM_SPRING";
    }
}
```

Even browser says **..Cross-Origin Request Blocked: The Same Origin Policy..** , look at the console of the spring application:

```wiki
2021-05-13 13:57:04.718  INFO 67306 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2021-05-13 13:57:04.725  INFO 67306 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 7 ms
Method called
```

That's means endpoint was actually called. Therefore CORS:

> **Does not do anything related to call of the endpoint. It just blocks the response to be accessed**
>
> **Does not protect the mutation operation.** In our case POST endpoint (mutation operation) was actually called, therefore something in the web application will be updated (database, cache etc ..)

## How to resolve Access-Control-Allow-Origin => Missing Error <a name="resolve_missing_error" />

There are many ways to resolve this error. I will just show two of them.

**NOTE: Please use one of the method, do not implement the both.**

### CORS support through Spring Security <a name="cors_for_spring_security" />

```java
@Component
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    @Bean
    public CorsFilter corsFilter() {
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        CorsConfiguration corsConfiguration = new CorsConfiguration();
        corsConfiguration.setAllowedHeaders(List.of("*"));
        corsConfiguration.setAllowedOrigins(Arrays.asList("*"));
        corsConfiguration.setAllowedMethods(Arrays.asList("*"));
        corsConfiguration.setMaxAge(Duration.ofMinutes(10));
        source.registerCorsConfiguration("/**", corsConfiguration);
        return new CorsFilter(source);
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable();
        http.authorizeRequests().anyRequest().permitAll();
        http.cors(); // add this line;
    }
}
```

### CORS Setup through MVC Application <a name="cors_for_mvc" />

Create a `WebMvcConfigurer` bean

```java
@Component
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/**").allowedOrigins("*");
            }
        };
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable();
        http.authorizeRequests().anyRequest().permitAll();
    }
}
```

...
