---
layout: post
title: "Aspect Oriented Programming(AOP) with Spring Boot - 1"
date: 2020-11-13 17:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

# Aspect Oriented Programming(AOP) with Spring Boot - 1

In this post, we are going to look at what is AOP and how your spring application matches with AOP concepts using Spring AOP module. After that we are going to look at the what is AspectJ, differences between Spring AOP and AspectJ.

> Note: Spring AOP Example in this topic is not the fully compliant example. I will hopefully write another post for only spring aop and aspectJ example.

Today topics are:

1. What is AOP and what problem AOP is trying to solve?
2. Terminology in AOP
3. How to use AOP in your Spring project via module called **Spring AOP**
4. Proxy Pattern
5. Spring AOP Example
6. What is AspectJ and AspectJ vs Spring AOP
   1. Weaving, Load Time Weaving and Compiler Time Weaving

## 1. What is AOP and What problem AOP is trying to solve?

- From the [wikipeadia](https://en.wikipedia.org/wiki/Aspect-oriented_programming) : **Aspect oriented programming** is a programming paradigm that aims to increase modularity by allowing the _separation of cross-cutting concerns._ It does so by adding additional behavior(an **advice**) to existing code without modifying the code itself.

> Note: Concern is a particular set of information that has an effect on the code of an computer programming. For example, business logic is a concern that effects our program execution.

Definition is easily to be understandable if we know what is **the cross-cutting concern**

### What are those cross-cutting concerns?

- **The cross-cutting concern** is a concern which can be applicable throughput the application and it effects the entire application.

It can be more understandable with examples:

- For instance, security is one of the cross-cutting concern. Let's you have decided that only one endpoint can be accessible without login requirement in your application. After that I assumed that you updated your application like this:

```java
public class UserLoginStatus{
  public static boolean isLoggedInUser(HttpServletRequest req){
    // returns true if user is logged in
  }
}
// ..
public class Controller{

  public String accessibleEndpoint(HttpServletRequest req){
   	return "success"
  }

  public String endpoint1(HttpServletRequest requestForEndpoint1){
    if (!UserLoginStatus.isLoggedInUser(..){
      return "error";
    }
    return "success"
  }

  public String endpoint2(HttpServletRequest requestForEndpoint2){
    if (!UserLoginStatus.isLoggedInUser(..)){
      return "error";
    }
    return "success"
  }

  public String endpoint3(HttpServletRequest requestForEndpoint3){
    if (!UserLoginStatus.isLoggedInUser(..)){
      return "error";
    }
    return "success"
  }
}
```

Solution from the above may seem the correct one, because you have only one method(`isLoggedInUser(..)`) that determines whether user is logged in or not. However, in that case, we are adding extra logic to our business which is not directly needed. In other words, for each business logic you have, you are extending with the security concern. At the end your business logic will be equal to **business logic + security concern**

- Let's say after that update, you decided to log all endpoints. Then I assumed that you updated your application like this one:

```java
public class UserLoginStatus{
  public static boolean isLoggedInUser(HttpServletRequest req){
    // return true if loggedin user for that request
  }
}
// ..
public class Controller{
  private static final Logger LOGGER = LoggerFactory...;

  public String accessibleEndpoint(HttpServletRequest req){
    final String methodName = "accessibleEndpoint";
    LOGGER.info("log for method: {} and reques: {} ", methodName, req);
   	return "success"
  }

  public String endpoint1(HttpServletRequest requestForEndpoint1){
    final String methodName = "endpoint1";
    LOGGER.info("log for method: {} and reques: {} ", methodName, req);
    if (!UserLoginStatus.isLoggedInUser(..){
      LOGGER.info("not logged in user");
      return "error";
    }
    LOGGER.info("another log")
    return "success"
  }

  public String endpoint2(HttpServletRequest requestForEndpoint2){
    final String methodName = ...
    LOGGER.info("log for method: {} and reques: {} ", methodName, req);
    if (!UserLoginStatus.isLoggedInUser(..)){
      return "error";
    }
    LOGGER.info("another log")
    return "success"
  }

  public String endpoint3(HttpServletRequest requestForEndpoint3){
    final String methodName = ...
    LOGGER.info("log for method: {} and reques: {} ", methodName, req);
    if (!UserLoginStatus.isLoggedInUser(..)){
      LOGGER.info("not logged in user");
      return "error";
    }
    LOGGER.info("another log")
    return "success"
  }
}
```

As you can see, right now for each business logic you have, you are extending with logging concern. At the end your business logic will be equal to **business logic + security concern + logging concern**

It is clear that solutions like the above are not the feasible. It would be nice if there is central class that handles all these concerns (which are not directly related to our business logic).

**That's the AOP is trying to solve. AOP allows centralized implementation of cross-cutting concerns. Because without AOP, they can not be implemented in a single place.**

## 2. Terminology in AOP

Actually all terms are related to each. So understand **what aspect does mean** will also allows us to understand other concepts as well.

- **Aspect**: The class that implements the cross cutting concern. For instance `LoggingAspect` implements logging feature for us.
  - Aspect contains **pointcut and advice**. In other words **Aspect = pointcut + advice**
- **Pointcut**: where the aspect is applied. (for instance, apply the logging aspect inside the controller package, but not repository package)
- **Advice**: what code to be executed.
- **Joinpoint**:
  - **Jointpoints** is the point in the control flow of a program. (Executing method for an example is a joinpoint)
  - To determine which method was called, we use **joinpoint**.
  - **Advices** can be presented with information about the joinpoint. For example: method name, class name etc.

There are more definitions than these. But for now let's look at the how we can implement aspect in spring framework.

## 3. Spring AOP

- It is another module from spring framework to work with aspect oriented programming.
- It is implemented in pure Java. There is no need for a special compilation process.
- Spring AOP currently supports only method execution join points (advising the execution of methods on Spring beans).

> Keep reading, you will understand what is method execution

- Spring AOP defaults to using standard J2SE _dynamic proxies_ for AOP proxies. This enables any interface (or set of interfaces) to be proxied.
- Spring's Proxy based AOP can only intercept `public` methods
- Also you can force the Spring AOP to use CGLIB proxies.

Before doing some examples, let's quick recap what is proxy pattern.

## 4. Proxy Pattern

- Proxy is a structural design pattern that allows us to create an intermediary object that acts as a real object. A proxy receives client requests, does some work such as access control, caching, security things and then passes the request to a real objects.

Let's say you have an service which returns user's image/icon from database. After some time, you have decided to cache those images to response as much as quickly. Before caching those requests, you have a code structure like this:

```java
public UserImageDTO{
    private Long id;
    private Image userImage;

}

public interface UserImageService{
    Image returnUserImageById(long id);
}

public class UserImageServiceImpl implements UserImageService{
    @Override
    public Image returnUserImageById(long id){
        // get userImageDTO from repository
        UserImageDTO userImageDTO = userRepository.findById(id);
        return userImageDTO.userImage();
    }
}

public class Client{
    UserImageService userImageService = new UserImageServiceImpl();
    Image userImage = userImageService.returnUserImageById(1);
}
```

After using proxy pattern for caching those requests:

```java
public UserImageServiceProxyImpl implements UserImageService{
    private UserImageService userImageService;
    private Map<Long, Image> cacheImages;

    public UserImageServiceProxyImpl(){
        cacheImages = new HashMap<>();
        this.userImageService = new UserImageServiceImpl();
    }

    @Override
    public Image returnUserImageById(long id){
        if (cacheImages.size() >= 1000){
            removeFirstElementFromCache();
        }

        if (cacheImages.get(id) == null){
            Image userImage = userImageService.returnUserImageById(id);
            addNewImageToCache(userImage);
            return userImage;
        }else{
            return cacheImages.get(id);
        }
    }
}

public class Client{
    UserImageService userImageService = new UserImageServiceProxyImpl();
    Image userImage = userImageService.returnUserImageById(1);
}
```

As you noticed, client does not know whether it is talking with proxy object or real service, what client gets back as a response is **user image instance.**

And using proxy, you are caching those images before returning to the client, and also there is second check. You are checking that your cache size is not grater than 1000.

Remember that, with these strategy you can also catch the errors from real object, or you can check the client's parameters etc..

**AOP does this in more concrete and dynamic way.**

## 5. Spring AOP Example with Spring Boot

Create simple spring boot project from Spring Initializr with these dependencies:

```xml
<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-aop</artifactId>
		</dependency>
</dependencies>
```

This simple project contains two controller classes:

```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String helloController(){
        return "Hello";
    }
}
// ...
@RestController
public class HomeController {
    @GetMapping("/")
    public String homeController(){
        return "Home";
    }
}
```

### Log every request inside the controller

For two controllers it is easy to do that without aop:

```java
@RestController
public class HelloController {
    private static final Logger LOGGER = LoggerFactory.getLogger(HelloController.class);

    @GetMapping("/hello")
    public String helloController(){
        final String methodName = "helloController";
        LOGGER.info("Log for method : {}", methodName);
        return "Hello";
    }
}

@RestController
public class HomeController {
    private static final Logger LOGGER = LoggerFactory.getLogger(HomeController.class);
    @GetMapping("/")
    public String homeController(){
        final String methodName = "home";
        LOGGER.info("Log for method : {}", methodName);
        return "Home";
    }
}
```

```wiki
2020-11-07 21:20:51.671  INFO 25655 --- [nio-8080-exec-1] c.s.s.controller.HelloController         : Log for method : helloController

2020-11-07 21:20:55.432  INFO 25655 --- [nio-8080-exec-2] c.s.springaop.controller.HomeController  : Log for method : home
```

### Log every request for method `homeController` with AOP

To create an aspect, you should follow these steps:

- Step 1: Create class annotated with `@Aspect`
- Step 2: Declare a pointcut (where the aspect is applied)
- Step 3: Declare an advice (code to be executed)

#### Creating an aspect for only homeContoller

- Step 1:

```java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Aspect
@Component // make sure that class is annotated with component, otherwise spring can not find your aspect
public class LoggingAspect {

}
```

- Step 2:

```java
@Aspect
@Component
public class LoggingAspect {
    @Before("execution(String homeController())") // pointcut expression
    private void loggingAspectForHomeController(){
    }
}
```

- Step 3:

```java
@Aspect
@Component
public class LoggingAspect {
    private static final Logger LOGGER = LoggerFactory.getLogger(LoggingAspect.class);

    @Before("execution(String homeController())")
    private void loggingAspectForHomeController(){
        // code to be executed
        LOGGER.info("aspect works");
    }
}
```

You have declared an aspect for your homeController. After that just run the project and browse the `http://localhost:8080/`:

```wiki
2020-11-07 21:33:54.854  INFO 26837 --- [nio-8080-exec-1] c.s.springaop.aspects.LoggingAspect      : aspect works

2020-11-07 21:33:54.863  INFO 26837 --- [nio-8080-exec-1] c.s.springaop.controller.HomeController  : Log for method : home
```

#### What was the `@Before` and `execution(...)` ?

`@Before` is one of the advice type and `execution(..)` part is the pointcut expression.

#### Advice Types

##### Before Advice (`@Before`)

- Executed before the actual method.
- If you want to avoid the call original method, you can do that only throwing exception inside the advice.
- And the end execution is propagated to the caller.

##### After Advice (`@After`)

- Executed after the original method

##### After Throwing (`@AfterThrowing` )

- Executed if the actual method throws an exception
- Exception will be propagated to the caller.

##### After Returning (`@AfterReturning`)

- Executed if the method returned successfully

##### Around (`@Around`)

- Wraps around the method
- Only the one who can prevent the original method from being called. (this can also be done by throwing exception at `@Before` advice, but at the end it will be propagated to the caller)
- Only advice that can catch exceptions and it will not propagated to the caller.
- Only advice that can modify return value.
- We use `ProceedingJoinPoint` which extends `JoinPoint` in that advice. This class includes method `proceed()`that allows us to continue the original method call.

#### Pointcut Expression

- This is the expression where aspect is applied. In the example above:

```java
// execution means that it is a method execution pointcut expression
// execute **before advice**, before the method execution called homeController which returns String and takes no parameter.
@Before("execution(String homeController())")
```

We can also use wild-cards for parts of the expression:

- Wild-card for method name : `*`
- Wild-card for return type: `*`
- Wild-card for parameters: `..`

```java
// expression for any method name which takes any parameter and any return type
"execution(* *(..))"
```

> There are also other pointcut expressions such as `within, args`. For more info you can check the [spring-aop-pointcut-tutorial](https://www.baeldung.com/spring-aop-pointcut-tutorial) from Bealdung.

#### Some pointcut expressions

- the execution of any method:

```java
"execution(* *(..))"
```

- the execution of any method with a name beginning with "set":

```java
"execution(* set*(..))"
```

- the execution of any method defined by the AccountService interface:

```java
"execution(* com.xyz.service.AccountService.*(..))"
```

- the execution of any method defined in the service package:

```java
"execution(* com.xyz.service.*.*(..))"
```

- the execution of any method defined in the service package or sub-package:

```java
"execution(* com.xyz.service..*.*(..))"
```

#### Updating current Aspect with `@After and @AfterReturning`

```java
@Aspect
@Component
public class LoggingAspect {
    private static final Logger LOGGER = LoggerFactory.getLogger(LoggingAspect.class);

    @Before("execution(String homeController())")
    private void loggingBeforeAspectForHomeController(){
        LOGGER.info("aspect BEFORE advice");
    }

    @After("execution(String homeController())")
    private void loggingAfterAspectForHomeController(){
        LOGGER.info("aspect AFTER advice");
    }

    @AfterReturning("execution(String homeController())")
    private void loggingAfterReturningAspectForHomeController(){
        LOGGER.info("aspect AFTER-RETURNING if calling method was finished successfully");
    }
}
```

Just run the project and hit the http://localhost:8080/

```wiki
2020-11-08 15:14:07.888  INFO 7847 --- [nio-8080-exec-1] c.s.springaop.aspects.LoggingAspect      : aspect BEFORE advice

2020-11-08 15:14:07.903  INFO 7847 --- [nio-8080-exec-1] c.s.springaop.controller.HomeController  : Log for method : home

2020-11-08 15:14:07.905  INFO 7847 --- [nio-8080-exec-1] c.s.springaop.aspects.LoggingAspect      : aspect AFTER-RETURNING if calling method was finished successfully

2020-11-08 15:14:07.905  INFO 7847 --- [nio-8080-exec-1] c.s.springaop.aspects.LoggingAspect      : aspect AFTER advice
```

#### Updating current Aspect with `@AfterThrowing`

If you want to see what was the exception inside the homeController, you can use `@AfterThrowing` . But be aware of the execution will be propagated to the caller:

First update the `homeController()`:

```java
@RestController
public class HomeController {
    private static final Logger LOGGER = LoggerFactory.getLogger(HomeController.class);

    @GetMapping("/")
    public String homeController(){
        final String methodName = "home";
        LOGGER.info("Log for method : {}", methodName);
        throw new RuntimeException("Exception inside home controller");
    }
}
```

Then update the `LoggingAspect`

```java
@Aspect
@Component
public class LoggingAspect {
    private static final Logger LOGGER = LoggerFactory.getLogger(LoggingAspect.class);

    @Before("execution(String homeController())")
    private void loggingBeforeAspectForHomeController(){
        LOGGER.info("aspect BEFORE advice");
    }

    @After("execution(String homeController())")
    private void loggingAfterAspectForHomeController(){
        LOGGER.info("aspect AFTER advice");
    }

    @AfterReturning("execution(String homeController())")
    private void loggingAfterReturningAspectForHomeController(){
        LOGGER.info("aspect AFTER-RETURNING if calling method was finished successfully");
    }

    @AfterThrowing(pointcut = "execution(String homeController())", throwing = "exception")
    private void homeExceptionAdvice(Throwable exception){
        LOGGER.error("aspect AFTER-THROWING: ", exception);
    }
}
```

Then after hit the localhost:8080

```wiki
2020-11-08 15:26:32.146  INFO 8401 --- [nio-8080-exec-1] c.s.springaop.aspects.LoggingAspect      : aspect BEFORE advice

2020-11-08 15:26:32.155  INFO 8401 --- [nio-8080-exec-1] c.s.springaop.controller.HomeController  : Log for method : home

2020-11-08 15:26:32.160 ERROR 8401 --- [nio-8080-exec-1] c.s.springaop.aspects.LoggingAspect      : aspect AFTER-THROWING:
java.lang.RuntimeException: Exception inside home controller
	at com.springexamples.springaop.controller.HomeController.homeController(HomeController.java:15) ~[classes/:na]
// ...

2020-11-08 15:26:32.160  INFO 8401 --- [nio-8080-exec-1] c.s.springaop.aspects.LoggingAspect      : aspect AFTER advice

2020-11-08 15:26:32.163 ERROR 8401 --- [nio-8080-exec-1] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.RuntimeException: Exception inside home controller] with root cause

java.lang.RuntimeException: Exception inside home controller
	at com.springexamples.springaop.controller.HomeController.homeController(HomeController.java:15) ~[classes/:na]
	at com.springexamples.springaop.controller.HomeController$$FastClassBySpringCGLIB$$c5b9afa9.invoke(<generated>) ~[classes/:na]
	// ...
```

As you can see from the above, first `@AfterThrowing` method will be executed, then execution will be propagated to the actual caller. That's why you will have two logs for the same exception.

#### Updating current Aspect with `@Around`

This one is the most powerful advice type. Let's catch the exception from the previous example. In that case execution will not be propagated to the caller.

```java
@Aspect
@Component
public class LoggingAspect {
    private static final Logger LOGGER = LoggerFactory.getLogger(LoggingAspect.class);

    @Around("execution(String homeController())")
    private Object loggingAroundAspectForHomeController(ProceedingJoinPoint proceedingJoinPoint) throws Throwable{
        LOGGER.info("aspect AROUND advice for signature name: {}", proceedingJoinPoint.getSignature().getName());
        try{
            return proceedingJoinPoint.proceed();
        }catch (Exception e){
            return "homeNotFound";
        }
    }
}
```

After hit the localhost:8080

```wiki
2020-11-08 15:34:13.725  INFO 8590 --- [nio-8080-exec-1] c.s.springaop.aspects.LoggingAspect      : aspect AROUND advice for signature name: homeController

2020-11-08 15:34:13.739  INFO 8590 --- [nio-8080-exec-1] c.s.springaop.controller.HomeController  : Log for method : home
```

That's it. No error log messages, and in the browser you will have seen **homeNotFound** response. Because `@Around` advice is the only advice that can change response of the calling method.

## What is AspectJ and AspectJ vs Spring AOP

- **AspectJ** is an implementation of aspect-oriented programming for java.

You most probably are interesting of AspectJ rather than Spring-AOP because:

- AspectJ is faster than Spring-AOP
- Aspect is more powerful. You can add more points in your program than Spring-AOP allows you to do.
- It is compatible with Spring-AOP.
  - Spring-AOP uses the same syntax with AspectJ
    - `@Aspect`, `@Before`, `@After` and `@Pointcut` works in the AspectJ also.
    - Pointcut expression that you wrote for Spring-AOP can also work with AspectJ.
    - **The only difference is that bean Pointcut expression won't work in AspectJ. Because bean pointcut expression depends on Spring Container**
- AspectJ uses **Bytecode weaving**. Means that both classes and aspect are **woven** into the Bytecode. AspectJ takes the aspects and the classes, mixes them together (called weaving them), then it is put into the bytecode.

> Bytecode is the code that is executed by the JVM(Java Virtual Machine)

Weaving can be done:

1. When classes are loaded (**Load Time Weaving**)
2. When code is compiled (**Compile Time Weaving**)

In general weaving can be defined in this picture:

<img src="/assets/spring/aop/aspectj.png" alt="aspectj" />

### Load Time Weaving

- It is a term for weaving happens when classes are loaded.
- In other words, **aspect are woven when classes are loaded**.

I am not going into detail of the configuration setup for load time weaving. You can learn more from this [link](https://www.eclipse.org/aspectj/doc/released/devguide/ltw-configuration.html). For instance, you need to enable **aspectj java agent** for load time weaving. That Java agents can modify the bytecode at load time.

### Compile Time Weaving

- It is a term for weaving happens when classes are compiled.
- In other words, **compiler weaves Aspects into the Bytecode**
- To allow compiler for weaving stuff, **you need to replace Java compiler with AspectJ compiler**

You can enable compile time weaving only updating the `pom.xml` . There is no need such as java agent.

And also I am not going into detail of the configuration setup for compile time weaving. You can look at the eclipse AspectJ developer guide from this [link](https://www.eclipse.org/aspectj/doc/released/) or you can search for also maven plugin `aspectj-maven-plugin`

There are so much things to write, but that's the end of this part, wait for the next one...
