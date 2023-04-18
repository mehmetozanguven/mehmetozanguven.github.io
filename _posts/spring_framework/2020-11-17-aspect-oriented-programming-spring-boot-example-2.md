---
layout: post
title: "Aspect Oriented Programming(AOP) with Spring Boot Example - 2"
date: 2020-11-17 12:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/spring-boot/aspect-oriented-programming-spring-boot-example-part-2"
---

# Aspect Oriented Programming(AOP) with Spring Boot Example - 2

This is the second post for a short series of **aspect oriented programming with the spring boot.**

In this post, I am going to implement a simple project example includes spring aop module. I will write aspect for both method execution(s) and annotation(s). At the end you will be able to reference this basic project for yours.

> Before start, you may need to check the previous [post](https://mehmetozanguven.github.io/spring/2020/11/13/aspect-oriented-programming-with-spring-boot-1.html) which includes AOP definitions, advice types, aspectJ etc..
>
> If you only need to use github repo, here is the [link](https://github.com/mehmetozanguven/spring-boot-examples/tree/master/spring-aop)

## Aspects in the Project

Before diving into the project, let me point to the aspects I am going to use for that project.

### @InjectResponseCookie( cookieValue = "value") Aspect

- This is a method level annotation.
- If any method inside the controller package has this annotation, then `value` will be added to the response cookie.

> In this example, I will generate random cookie name with the static cookie value, therefore you can verify that aspect is working by logging them on the controller

### @Retry Aspect

- This is a also method level annotation
- If any method has this annotation, then Retry aspect will re-execute the method again.

### Logging Aspect

- For any execution of the `POST` method, log that "POST method called"

### Tricky Method Aspect

- **Will aspect work when you call it from another method e.g indirect-call ?**
- **How Spring AOP actually works**

## Basic Project Structure

Before adding the aspect, default project setup includes three controllers (`CustomerController, LoginController, StatusController`) and two services (`CustomerService, LoginService`) and one repository (`CustomerRepository`)

There is no real database setup or other complex setup to start this application. These are just the dummy endpoints. Take a look at all the endpoints:

```java
package com.mehmetozanguven.springaopexample;

@SpringBootApplication
public class SpringAopExampleApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringAopExampleApplication.class, args);
	}
}
// ...
package com.mehmetozanguven.springaopexample.controller;

@RestController
@RequestMapping("/api")
public class StatusController {

    @GetMapping("/status")
    public String getApplicationStatus(){
        return "application-is-working";
    }
}
// ...
package com.mehmetozanguven.springaopexample.controller;

@RestController
@RequestMapping("/api")
public class LoginController {

    @Autowired
    private LoginService loginService;

    @PostMapping("/login")
    public String loginCustomer(LoginRequest loginRequest){
        loginService.loginCustomer(loginRequest);
        return "login-request";
    }
}
// ...
package com.mehmetozanguven.springaopexample.controller;

@RestController
@RequestMapping("/api")
public class CustomerController {
    @Autowired
    private CustomerService customerService;

    @GetMapping("/customer-by-id/{id}")
    public String findCustomerById(HttpServletResponse response, @PathVariable String id){
        customerService.findCustomerById(Long.parseLong(id));
        return "customer-by-id-" + id;
    }
}
```

## Adding Aspect

Before directly adding annotation to other project,

First define some pointcut designators:

- **execution**: matching method execution join points. This is the most widely used.
- **within**: for matching methods of classes within certain types e.g. classes within a package.
- **@within** – for matching to join points within types (target object class) that have the given annotation. (used for class level annotation)
- **@annotation –** for matching to join points where the subject (method) of the Joinpoint has the given annotation. (method level annotation)

Second define some _pre-define_ pointcut, such as **pointcut for serviceClassMethod(s), repositoryClassMethod(s) etc..**

```java
@Aspect
public class SystemPointcut {
	// define pointcut for any method execution inside package controller and its sub-package
    // we can refer this pointcut via controllerLayer()
    @Pointcut("within(com.mehmetozanguven.springaopexample.controller..*)")
    public void controllerLayer() {}

    // ...
}

```

### @InjectResponseCookie aspect

Define the annotation:

```java
package com.mehmetozanguven.springaopexample.annotation;

public @interface InjectResponseCookie {
    String cookieValue() default "defaultValue";
}
```

Define the aspect: **I decided to use `@After` advice for that. I can also use this `@Around` however this aspect will not modify the return value of the actual method execution.**

```java
@Aspect
@Component
public class InjectResponseCookieAspect {
    private static final Logger LOGGER = LoggerFactory.getLogger(InjectResponseCookieAspect.class);

    @After(value = "com.mehmetozanguven.springaopexample.aspect.SystemPointcut.controllerLayer() && " +
            "@annotation(injectResponseCookie)")
    public void injectResponseCookie(InjectResponseCookie injectResponseCookie){
        HttpServletResponse response = getResponse();
        Cookie cookie = new Cookie(generateRandomCookieName(), injectResponseCookie.cookieValue());
        response.addCookie(cookie);
        LOGGER.info("Annotation value: {}", injectResponseCookie.cookieValue());
    }
}
```

Add `@InjectResponseCookie` annotation to any method: (I have added the `findCustomerById` method)

```java
@InjectResponseCookie(cookieValue = "injectResponseCookie")
@GetMapping("/customer-by-id/{id}")
public String findCustomerById(HttpServletRequest request, HttpServletResponse response, @PathVariable String id) {
  customerService.findCustomerById(Long.parseLong(id));
  logAllCookies(request);
  return "customer-by-id-" + id;
}
// hit the http://localhost:8080/api/customer-by-id/22 and see the cookies
```

### @Retry Aspect

Define the annotation:

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Retry {
}
```

Define the aspect: **This should be `@Around` advice, because i need to catch the exception and modify the calling producure, **

> Note: `@Around` advice is the only advice that can prevent the original method from being called and only advice that can catch exceptions and it will not propagated to the caller. For more info about advice types check out my previous [post](https://mehmetozanguven.github.io/spring/2020/11/13/aspect-oriented-programming-with-spring-boot-1.html)

```java
@Aspect
@Component
public class RetryAspect {
    private static final Logger LOGGER = LoggerFactory.getLogger(RetryAspect.class);

	// Fully qualified name to avoid 'error Type referred to is not an annotation type:'
    @Around("@annotation(com.mehmetozanguven.springaopexample.annotation.Retry)")
    public Object retryTheExecution(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        try{
            return proceedingJoinPoint.proceed();
        }catch (Throwable throwable){
            LOGGER.error("Error when calling method: {}, retrying again...", proceedingJoinPoint.getSignature().getName());
            return proceedingJoinPoint.proceed();
        }
    }
}
```

Add `@Rety` to any method execution, I have added to the `StatusController`, logic is very basic:

```java
@RestController
@RequestMapping("/api")
public class StatusController {
    private static Logger LOGGER = LoggerFactory.getLogger(StatusController.class);

    @Retry
    @GetMapping("/status")
    public String getApplicationStatus(){
        int randomNumber = getRandomNumber();
        LOGGER.info("Status controller with randomNumber: {}", randomNumber);
        if (randomNumber % 2 != 0){
            throw new RuntimeException("Dummy exception to test retry");
        }
        return "application-is-working";
    }

    private int getRandomNumber(){
        return RandomUtils.nextInt(0, 10);
    }
}
```

Depending on your execution, you may see the logs like this:

```wiki
2020-11-17 00:08:52.802  INFO 15258 --- [nio-8080-exec-1] c.m.s.controller.StatusController        : Status controller with randomNumber: 5

2020-11-17 00:08:52.804 ERROR 15258 --- [nio-8080-exec-1] c.m.springaopexample.aspect.RetryAspect  : Error when calling method: getApplicationStatus, retrying again...

2020-11-17 00:08:52.804  INFO 15258 --- [nio-8080-exec-1] c.m.s.controller.StatusController        : Status controller with randomNumber: 8
```

### Logging Aspect

Define the aspect:

```java
@Aspect
@Component
public class PostLoggingAspect {
    private static final Logger LOGGER = LoggerFactory.getLogger(PostLoggingAspect.class);

    @Before("@annotation(org.springframework.web.bind.annotation.PostMapping)")
    public void logBeforePostMethod(JoinPoint joinPoint){
        LOGGER.info("POST method called: {}", joinPoint.getSignature());
    }
}
```

You can hit the login endpoint via this payload

```json
{
  "email": "sample",
  "password": "password"
}
```

Here is the aspect log:

```wiki
2020-11-17 00:18:15.420  INFO 15979 --- [nio-8080-exec-1] c.m.s.aspect.PostLoggingAspect           : POST method called: String com.mehmetozanguven.springaopexample.controller.LoginController.loginCustomer(LoginRequest)
```

### Tricky Method Aspect

As I have said previously, We are going to find out "aspect will work for indirect call or not?"

Define aspect:

```java
@Aspect
@Component
public class TrickyAspect {
    private static final Logger LOGGER = LoggerFactory.getLogger(TrickyAspect.class);

    @After("execution(* trickyMethod(..))")
    public void trickyMethodAdvice(){
        LOGGER.info("Tricky after advice called");
    }
}
```

Update the `CustomerController,Service and Repository`

```java
@RestController
@RequestMapping("/api")
public class CustomerController {
	// ...
    @GetMapping("/tricky")
    public String callTrickyDirectly(){
        customerService.callTrickyMethodDirectly();
        return "direct call";
    }

    @GetMapping("/tricky-in")
    public String callTrickyInDirectly(){
        customerService.callingTrickyMethodInDirectly();
        return "indirect call";
    }
}
```

```java
@Service
public class CustomerServiceImpl implements CustomerService{
	// ...
    @Override
    public void callTrickyMethodDirectly() {
        customerRepository.trickyMethod();
    }

    @Override
    public void callingTrickyMethodInDirectly() {
        customerRepository.indirectCallOfTrickyMethod();
    }
}
```

```java
@Repository
public class CustomerRepositoryImpl implements CustomerRepository {
    private static final Logger LOGGER = LoggerFactory.getLogger(CustomerRepositoryImpl.class);

    @Override
    public void trickyMethod() {
        LOGGER.info("Tricky method called");
    }

    @Override
    public void indirectCallOfTrickyMethod() {
        LOGGER.info("-------");
        LOGGER.info("indirectCallOfTrickyMethod will call the trickyMethod");
        LOGGER.info("Aspect will not work in that case.");
        LOGGER.info("-------");
        trickyMethod();
    }
}
```

Hit the: http://localhost:8080/api/tricky, here is the logs:

```wiki
2020-11-17 01:14:15.911  INFO 20684 --- [nio-8080-exec-1] c.m.s.repository.CustomerRepositoryImpl  : Tricky method called

2020-11-17 01:14:15.912  INFO 20684 --- [nio-8080-exec-1] c.m.s.aspect.TrickyAspect                : Tricky after advice called
```

Hit the http://localhost:8080/api/tricky-in, here is the logs:

```wiki
2020-11-17 01:16:28.204  INFO 20927 --- [nio-8080-exec-1] c.m.s.repository.CustomerRepositoryImpl  : -------

2020-11-17 01:16:28.204  INFO 20927 --- [nio-8080-exec-1] c.m.s.repository.CustomerRepositoryImpl  : indirectCallOfTrickyMethod will call the trickyMethod

2020-11-17 01:16:28.204  INFO 20927 --- [nio-8080-exec-1] c.m.s.repository.CustomerRepositoryImpl  : Aspect will not work in that case.

2020-11-17 01:16:28.204  INFO 20927 --- [nio-8080-exec-1] c.m.s.repository.CustomerRepositoryImpl  : -------

2020-11-17 01:16:28.204  INFO 20927 --- [nio-8080-exec-1] c.m.s.repository.CustomerRepositoryImpl  : Tricky method called
```

As you can see TrickyAfterAdvice did not get call. If you look for an answer, answer is related to the Spring proxy mechanism.

When Spring knows your object (that's means you annotated your object with `@Component, @Service, @Conroller etc..`, what happens is that Spring wraps the original object via proxy. (Actually proxy pattern is applying here, for more information about proxy pattern, please look at the my previous post.)

All Spring beans communicate each other using the proxy. Because I am using Spring AOP, aspects is also known by Spring Container, therefore all advices method is being called with using the proxy object.

Here is your Original Object:

<img src="/assets/spring/aop/originalObject.png" alt="original_object" />

If Spring knows your object, Spring will wrap (will create a proxy) the original Object with proxy, this proxy looks like original object(s) and **this is injected into other spring beans**:

<img src="/assets/spring/aop/proxy.png" alt="original_object" />

Other spring beans does call to this proxy, this call is forwarded to the Original Object and also the Advice is called. This is **how Spring AOP works.** Instead of the Original object proxy is used:

<img src="/assets/spring/aop/adviceCall.png" alt="original_object" />

So what happens when you do a local method call (kind of a situation in the example "`indirectCallOfTrickyMethod()` have called the `trickyMethod()`")?

- In that case original object calls itself and proxy is never executed, that means Advice will never be executed. In other words, calls never reaches the Proxy.

<img src="/assets/spring/aop/localMethod.png" alt="original_object" />

We can see the local method call and proxy call adding debug point. Now just add a debug point in here:

```java
@Repository
public class CustomerRepositoryImpl implements CustomerRepository {
    private static final Logger LOGGER = LoggerFactory.getLogger(CustomerRepositoryImpl.class);
	// ...
    @Override
    public void trickyMethod() {
        LOGGER.info("Tricky method called");
    } // add debug point in here
	// ...
}
```

After hit the http://localhost:8080/api/tricky, you can see that `trickyMethod` will be called from class named: `CustomerRepositoryImpl$$FastClassBySpringCGLIB...` which is nothing but a Proxy, and that means Advice will be called also.

<img src="/assets/spring/aop/proxyCalled.png" alt="original_object" />

After hit the http://localhost:8080/api/tricky-in, `indirectCallOfTrickyMethod()` will be called by the same Proxy class from the above, but `trickyMethod()` call inside the `indirectCallOfTrickyMethod` will be done by the object itself. Therefore Proxy will never be executed and advice will not be called.

<img src="/assets/spring/aop/localMethodCall.png" alt="original_object" />

Another example could `@Transactional` annotation:

```java
@Repository
public class CustomerRepository{
    @Transactional
    public void transaction(){

    }

    public void callTransactional(){
        /* Because @Transaction is implemented using Spring AOP,
        	callTransactional method has no configuration(even it is calling transaction method) for transaction such as:
        		the rollback rules, timeout, isolation level etc..
        */
        transaction()
    }
}
```

That's it. I hope this post could be helpful for anyone that needs to use Spring-AOP.

Last but not least, wait it for the next one.
