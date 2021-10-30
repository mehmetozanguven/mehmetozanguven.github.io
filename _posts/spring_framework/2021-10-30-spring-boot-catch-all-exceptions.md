---
layout: post
title: Spring Boot how to catch all exceptions
date: 2021-10-39 12:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

In this quick tutorial, let's learn how to catch all exception(s) in Spring Boot. And most probably you want to return customized exception when something wrongs in your web application. Even if there are many ways to implement this feature, in this blog I am going to talk about the solution using `@ControllerAdvice`

## What is the ControllerAdvice in Spring?

A controller advice allows us to handle exception(s) across the whole application, not just to an individual controller.

In other words, it is the one global class to handle all exception(s)

To define Controller advice, you should annotate your class with `@ControllerAdvice`

## Example

Here is the practical example.

```java
@Order(Ordered.HIGHEST_PRECEDENCE)
@ControllerAdvice
public class MyExceptionHandler  {
    private static final Logger logger = LoggerFactory.getLogger(MyExceptionHandler.class);

    @ExceptionHandler(EmptyIpException.class)
    public ResponseEntity<?> emptyIpException() {
        ServiceGenericResponse<String> response =
                new ServiceGenericResponse.Builder<String>()
                        .isSuccess(false)
                        .httpStatus(HttpStatus.BAD_REQUEST)
                        .errorCode("1")
                        .errorMessage("Empty Ip Address")
                        .build();
        return new ResponseEntity<>(response, response.getHttpStatus());
    }
}



public class EmptyIpException extends RuntimeException {

    public EmptyIpException(String message) {
        super(message);
    }
}
```

- `@ExceptionHandler(EmptyIpException.class)` tells that we are going to handle `EmptyIpException`

> `ServiceGenericResponse`is a utility class define by me (not specific to Spring). Purpose of this class is to return well formatted exceptions and responses. You can ignore it.

Let's say in the controller I am intentionally throwing this exception:

 ```java
 @RestController
 @RequestMapping("/api")
 public class MyController {
 
     @GetMapping( "/ip/{ipAddress}")
     public ResponseEntity<?> getIpLocation(@PathVariable("ipAddress") Optional<String> pathIp) {
         
         throw new EmptyIpException("Ip can not be empty");
     }
 }
 ```

Now if I send this request: `curl -X GET localhost:8080/api/ip/1.1.1.1`, response will be:

```json
{
  "returnedObject": null,
  "errorCode": "1",
  "errorMessage": "Empty Ip Address",
  "httpStatus": "BAD_REQUEST",
  "httpStatusCode": 400,
  "success": false
}
```

## Catch NoHandlerFoundException

If you send this request: `curl -X GET localhost:8080/api/test/1.1.1.1`, then you will get  `NoHandlerFoundException`, because there is no controller for that request. To catch `NoHandlerFoundException`we need to setup extra things. Because, in default this exception will be handled by `DefaultHandlerExceptionResolver`. 

To handle this exception in our global `@ControllerAdvice`, please set the following properties (in the application.properties file):

```properties
spring.mvc.throw-exception-if-no-handler-found=true
spring.web.resources.add-mappings=false
```

Here is our customize exception for `NoHandlerFoundException`:

```java
@Order(Ordered.HIGHEST_PRECEDENCE)
@ControllerAdvice
public class MyExceptionHandler  {

    @ExceptionHandler(EmptyIpException.class)
    public ResponseEntity<?> emptyIpException() {
        ServiceGenericResponse<String> response =
                new ServiceGenericResponse.Builder<String>()
                        .isSuccess(false)
                        .httpStatus(HttpStatus.BAD_REQUEST)
                        .errorCode("1")
                        .errorMessage("Empty Ip Address")
                        .build();
        return new ResponseEntity<>(response, response.getHttpStatus());
    }
    
    @ExceptionHandler(NoHandlerFoundException.class)
    public ResponseEntity<?> noHandlerFoundException(Exception ex){
        ServiceGenericResponse<String> response =
                new ServiceGenericResponse.Builder<String>()
                        .isSuccess(false)
                        .httpStatus(HttpStatus.BAD_REQUEST)
                        .errorCode("7")
                        .errorMessage("No handler found")
                        .build();
        return new ResponseEntity<>(response, response.getHttpStatus());
    }
}
```

Right now, if I send this request: `curl -X GET localhost:8080/api/test/1.1.1.1`, then response will be:

```json
{
  "returnedObject": null,
  "errorCode": "7",
  "errorMessage": "No handler found",
  "httpStatus": "BAD_REQUEST",
  "httpStatusCode": 400,
  "success": false
}
```



## Catch all un-expected exceptions

We can also write a handler for all unexpected exceptions:

```java
@Order(Ordered.HIGHEST_PRECEDENCE)
@ControllerAdvice
public class MyExceptionHandler  {

    @ExceptionHandler(EmptyIpException.class)
    public ResponseEntity<?> emptyIpException() {
        ServiceGenericResponse<String> response =
                new ServiceGenericResponse.Builder<String>()
                        .isSuccess(false)
                        .httpStatus(HttpStatus.BAD_REQUEST)
                        .errorCode("1")
                        .errorMessage("Empty Ip Address")
                        .build();
        return new ResponseEntity<>(response, response.getHttpStatus());
    }
    
    @ExceptionHandler(NoHandlerFoundException.class)
    public ResponseEntity<?> noHandlerFoundException(Exception ex){
        ServiceGenericResponse<String> response =
                new ServiceGenericResponse.Builder<String>()
                        .isSuccess(false)
                        .httpStatus(HttpStatus.BAD_REQUEST)
                        .errorCode("7")
                        .errorMessage("No handler found")
                        .build();
        return new ResponseEntity<>(response, response.getHttpStatus());
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<?> genericException(Exception ex) {
        logger.error("Unexpected error has happened", ex);
        ServiceGenericResponse<String> response =
                new ServiceGenericResponse.Builder<String>()
                        .isSuccess(false)
                        .httpStatus(HttpStatus.INTERNAL_SERVER_ERROR)
                        .errorCode("6")
                        .errorMessage("unexpected error")
                        .build();
        return new ResponseEntity<>(response, response.getHttpStatus());
    }
}
```

 Let's throw RuntimeException in the controller:

```java
@RestController
@RequestMapping("/api")
public class MyController {
  	@GetMapping( "/ip/{ipAddress}")
    public ResponseEntity<?> getIpLocation(@PathVariable("ipAddress") Optional<String> pathIp) {
   
        throw new RuntimeException("unexpected error");
    }
}
```

Now, If I send this request: `curl -X GET localhost:8080/api/ip/1.1.1.1`, response will be:

```json
{
  "returnedObject": null,
  "errorCode": "6",
  "errorMessage": "unexpected error",
  "httpStatus": "INTERNAL_SERVER_ERROR",
  "httpStatusCode": 500,
  "success": false
}
```



If you want to know other ways to handle exception in Spring framework, you can click: [https://spring.io/blog/2013/11/01/exception-handling-in-spring-mvc](https://spring.io/blog/2013/11/01/exception-handling-in-spring-mvc)