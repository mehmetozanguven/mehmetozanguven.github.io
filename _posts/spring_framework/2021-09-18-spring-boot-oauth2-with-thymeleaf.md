---
layout: post
title: Spring Boot OAuht2 Implementation with Thymeleaf
date: 2021-09-18 22:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

In this article, we will implement a basic single sign on application using Spring boot. We will use Thymeleaf for html pages

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

A single sign-on (SSO) application is one in which we authenticate through an authorization server, and then the app keeps we logged-in, using a refresh token.

In our application we will use Google as the authorization and resource servers

<img src="/assets/spring/oauth2/thymeleaf/google_sign_on.png" alt = "google_sign_on.png"/>

> In this diagram:
>
> **User = resource owner**
>
> **Spring Boot Application = The Client**
>
> **Google = Authorization Server & Resource Server**

If you remember the my previous blog, the client will only needs to send client ID and client secret. Then, Google can identify our web application(the client). And also our client won't manage its users.

I will not go into the details of how to setup your google (via console.cloud.google.com) to use as OAuth. There are dons of tutorial on Internet.

> Just make sure that:
>
> - **Authorized Javascript origins** = http://localhost:8080 (The HTTP origins that host your web application.)
> - **Authorized redirect URIs** = http://localhost:8080/login/oauth2/code/google (Users will be redirected to this path after they have authenticated with Google.)

At the end google will prompt us a message:

<img src="/assets/spring/oauth2/thymeleaf/google_client_id.png" alt ="google_client_id.png"/>

> Note: Google restricts us to use only test users.

This configuration is everything we need from AuthorizationServer. We can implement the spring boot application.

## Dependencies

Create the spring boot application with the following dependencies:

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

## Create controller

We will return basic html page from the controller:

```java
@Controller
public class DemoController {

    @GetMapping("/")
    public String getHomePage() {
        return "homePage";
    }
}
```

Also creates `homePage.html` under the resources -> templates:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>HomePage</title>
  </head>
  <body>
    <h3>Welcome</h3>
  </body>
</html>
```

## Security Configuration Setup

Now, we can write our security configuration extending the class `WebSecurityConfigurerAdapter`. In this time, we will not use `formLogin()` we will use something different called `oauth2login()`

```java
@Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.oauth2Login(); // Set the authentication method

        http.authorizeRequests() // Set the all endpoints will be protected
                .anyRequest()
                .authenticated();
    }
}
```

- `oauth2login()` simply adds new authentication filter called `OAuth2LoginAuthenticationFilter`. (responsible for authentication)
- `oauth2login()` also adds another filter called `OAuth2AuthorizationRequestRedirectFilter`. (responsible for redirection of the endpoint of the authorization server)

> I have also written about the spring security filter chains and also created custom filter in the spring application. Link => [https://mehmetozanguven.github.io/spring/2020/12/30/spring-security-5-custom-filter.html](https://mehmetozanguven.github.io/spring/2020/12/30/spring-security-5-custom-filter.html)

- By calling `oauth2Login()` , we added the `OAuth2LoginAuthenticationFilter` which applies the needed logic for OAuth2 authentication

<img src="/assets/spring/oauth2/thymeleaf/oauth2_filter.png" alt="oauth2_filter.png" />

If we start application at this point, application will fail. Because we didn't specify any Authorization Server. For this we will use `ClientRegistration` interface.

## ClientRegistration

This interface represents the client in the OAuth2. For the client, we need to define:

- The client ID and secret
- The grant type
- The redirect URI
- The scopes

Here is the client registration for google:

```java
private ClientRegistration googleClientRegistration() {
        return ClientRegistration.withRegistrationId("google")
                .clientId("your_client_id")
                .clientSecret("your_client_secret")
                .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
                .redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
                .scope("openid", "profile", "email")
                .authorizationUri("https://accounts.google.com/o/oauth2/v2/auth")
                .tokenUri("https://www.googleapis.com/oauth2/v4/token")
                .userInfoUri("https://www.googleapis.com/oauth2/v3/userinfo")
                .userNameAttributeName(IdTokenClaimNames.SUB)
                .jwkSetUri("https://www.googleapis.com/oauth2/v3/certs")
                .clientName("Google").build();
    }
```

- **Authorization URI:** The URI to which the client redirects the user for authentication.
- **Token URI**: The URI that the client calls to obtain an access token and a refresh token. (If you don't understand what it's means, please read the [https://mehmetozanguven.github.io/spring/2021/09/15/spring-with-oauth2-theory.html](https://mehmetozanguven.github.io/spring/2021/09/15/spring-with-oauth2-theory.html))
- **User info URI**: The URI that the client call after obtaining an access token to get more details about the user (If you don't understand what it's means, please read the [https://mehmetozanguven.github.io/spring/2021/09/15/spring-with-oauth2-theory.html](https://mehmetozanguven.github.io/spring/2021/09/15/spring-with-oauth2-theory.html))

These URIs is defined by the Google itself.

But as you can see there are many fields to be filled. To avoid fill all the fields, Spring Security provides an easy way. Spring security provides an ClientRegistration builder for the most common provides:

- Google
- Facebook
- Github
- Okta

Then we can use the builder for Google:

```java
 private ClientRegistration googleClientRegistration() {
        return  CommonOAuth2Provider.GOOGLE
                .getBuilder("google")
                .clientId("your_client_id")
                .clientSecret("your_client_secret")
                .build();
    }
```

Creating clientRegistration is not enough, we should also tell the spring boot application to use it. To do that we will use `ClientRegistrationRepository`

## ClientRegistrationRepository

To setup ClienRegistration, Spring Security uses `ClientRegistrationRepository`

<img src="./pictures/client_registration_repository.png" alt="client_registration_repository.png" />

> Actually `ClientRegistrationRepository` is similar to `UserDetailsService`. As you recall `UserDetailsService` will try to find `UserDetails`. We can apply the same login in here: `ClientRegistrationRepository` tries to find `ClientRegistration`

We can use `InMemoryClientRegistrationRepository` (kind of `InMemoryUserDetailsService`) and put the google client inside it:

```java
Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    ....
    @Bean
    public ClientRegistrationRepository clientRepository() {
        ClientRegistration google = googleClientRegistration();
        return new InMemoryClientRegistrationRepository(google);
    }

    private ClientRegistration googleClientRegistration() {
        return  CommonOAuth2Provider.GOOGLE
                .getBuilder("google")
                .clientId("CLIENT_ID")
                .clientSecret("CLIENT_SECRET")
                .build();
    }
}
```

Note: we can also register the (instead of Bean) `ClientRegistrationRepository` using customized inside `http.oauh2Login()`

> ```java
> @Configuration
> public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
>     @Override
>     protected void configure(HttpSecurity http) throws Exception {
>         http.oauth2Login( custo ->
>                          custo.clientRegistrationRepository(clientRepository())
>                         ); // Set the authentication method
>
>         http.authorizeRequests() // Set the all endpoints will be protected
>                 .anyRequest()
>                 .authenticated();
>     }
>
>     public ClientRegistrationRepository clientRepository() {
>         ClientRegistration google = googleClientRegistration();
>         return new InMemoryClientRegistrationRepository(google);
>     }
>
>     private ClientRegistration googleClientRegistration() {
>         return  CommonOAuth2Provider.GOOGLE
>                 .getBuilder("google")
>                 .clientId(CLIENT_ID)
>                 .clientSecret(CLIENT_SECRET)
>                 .build();
>     }
> }
> ```

## Using properties file instead of ClientRegistration and Repository

Spring Boot will also allow us to use properties file to construct `ClientRegistration` and `ClientRegistrationRepository` instead of directly creating it. (Probably, you have seen this in many articles !!!)

Just add the following properties in the `application.properties` files:

```properties
spring.security.oauth2.client.registration.google.client-id=your_client_id
spring.security.oauth2.client.registration.google.client-secret=your_secret_key
```

Because we wrote `registration.google`, spring will automatically create a google client registration from the `CommonOAuth2Provider.GOOGLE`

After adding properties: we can remove the `ClientRegistrationRepository` bean and the method `googleClientRegistration()` from the configuration.

> If you need to load client registration from the database, you may use the Bean solution, otherwise using properties will be more cleaner

## How to get logged-in user

As you remember from the previous blog, logged-in user will be stored in the `SecurityContext`. We will use the `SecurityContext` to get logged-in user information when we also use `oauth2Login()`

```java
@Controller
public class DemoController {
    private static final Logger logger = LoggerFactory.getLogger(DemoController.class.getSimpleName());

     @GetMapping("/")
    public String getHomePage() {
        SecurityContext securityContext = SecurityContextHolder.getContext();
        OAuth2AuthenticationToken loggedInUser =(OAuth2AuthenticationToken) securityContext.getAuthentication();
        logger.info("Logged-in user: {}", loggedInUser);
        return "homePage";
    }
}
```

After you run the project, you will be redirected to the Google login page, write your gmail account and password, after that you will the homePage, and in the console:

```wiki
INFO 17474 --- [nio-8080-exec-6] DemoController                           : Logged-in user: OAuth2AuthenticationToken [Principal=Name: [...], Granted Authorities: [[ROLE_USER, SCOPE_https://www.googleapis.com/auth/userinfo.email, SCOPE_https://www.googleapis.com/auth/userinfo.profile, SCOPE_openid]], User Attributes: [{..., name=Mehmet Ozan Güven, email=mehmetozanguven@gmail.com}], Credentials=[PROTECTED], Authenticated=true, Details=WebAuthenticationDetails [RemoteIpAddress=127.0.0.1, ...], Granted Authorities=[ROLE_USER, SCOPE_https://www.googleapis.com/auth/userinfo.email, SCOPE_https://www.googleapis.com/auth/userinfo.profile, SCOPE_openid]]
```

## Conclusion

In this small spring boot application, we learned how to use OAuth2 with Spring Boot.

- When you call `http.oauth2Login()`, you are actually creating new filter called `OAuth2LoginAuthenticationFilter` and this filter will run its login in the security filter chains.
- Creating `OAuth2LoginAuthenticationFilter` is not enough, you should also create `ClientRegistration` and `ClientRegistrationRepository` to work with OAuth2
- `ClientRegistrationRepository` holds the `ClientRegistration`(s)
- You can also use properties, instead of directly creating `ClientRegistrationRepository` & `ClientRegistration`. For example, If you use common provider like Google, then you need two properties:

```properties
spring.security.oauth2.client.registration.google.client-id=your_client_id
spring.security.oauth2.client.registration.google.client-secret=your_secret_key
```

- You can get the Authenticated object from `SecurityContext`
