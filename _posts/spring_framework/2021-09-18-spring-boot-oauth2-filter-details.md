---
layout: post
title: Spring Boot OAuht2 Reason for /oauth2/authorization/{id}
date: 2021-09-18 23:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

Do you know the reason why redirect URL should be `http://localhost:8080/login/oauth2/code/google` when we setup Authorization Server in the console.cloud.google.com? Let's answer this question.

Also before doing an example with any Javascript framework it is good to know what it is going on when we we open localhost:8080 in the previous example => [https://mehmetozanguven.github.io/spring/2021/09/18/spring-boot-oauth2-with-thymeleaf.html](https://mehmetozanguven.github.io/spring/2021/09/18/spring-boot-oauth2-with-thymeleaf.html)

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

I have changed the previous example a little bit:

- First I have created the login page as accessible for anyone with the login link

```java
@Controller
public class DemoController {
    private static final Logger logger = LoggerFactory.getLogger(DemoController.class.getSimpleName());

    @GetMapping("/loggedIn")
    public String getHomePage() {
        // only logged-in user can access
        SecurityContext securityContext = SecurityContextHolder.getContext();
        OAuth2AuthenticationToken loggedInUser =(OAuth2AuthenticationToken) securityContext.getAuthentication();
        logger.info("Logged-in user: {}", loggedInUser);
        return "homePage";
    }

    @GetMapping("/")
    public String getLoginPage(){
        // everyone can access
        return "loginPage";
    }
}
```

Here is the `loginPage.html`:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Login With Google</title>
  </head>
  <body>
    <h1>Login with google</h1>
    <div>
      <a style="font-size:2rem" href="/oauth2/authorization/google"
        >Login With Google</a
      >
    </div>
  </body>
</html>
```

Also updated the `SecurityConfiguration`

```java
@Configuration
@EnableWebSecurity(debug = true) // do not enable debug in the production
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    String CLIENT_ID = "your_client_id";
    String CLIENT_SECRET = "your_client_secret";

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().antMatchers("/").permitAll(); // anyone can access the login page
        http.oauth2Login();

        http.authorizeRequests() // Set the all endpoints will be protected
                .anyRequest()
                .authenticated();
    }

    @Bean
    public ClientRegistrationRepository clientRepository() {
        ClientRegistration google = googleClientRegistration();
        return new InMemoryClientRegistrationRepository(google);
    }

    private ClientRegistration googleClientRegistration() {
        return  CommonOAuth2Provider.GOOGLE
                .getBuilder("google")
                .clientId(CLIENT_ID)
                .clientSecret(CLIENT_SECRET)
                .build();
    }
}
```

After running the project, If I open the http://localhost:8080

<img src="/assets/spring/oauth2/details/login_page.png" alt="login_page.png" />

And I try to go http://localhost:8080/loggedIn , I will be redirected to the Sign In with Google Page:

<img src="/assets/spring/oauth2/details/sign_with_google.png" alt="sign_with_google.png"/>

For successful authentication, I can access to the http://localhost:8080/loggedIn

<img src="/assets/spring/oauth2/details/welcome.png" alt="welcome.png"/>

The first question you may ask **why I created `<a>` html tag with the link `http://localhost:8080/oauth2/authorization/google`**

## Reason for `/oauth2/authorization/google`

As you recall from the previous Spring security series, Spring security is basically bunch of a filters applied by one by. (Order is crucial) And when we say:

```java
http.oauth2Login();
```

We are adding two filters in the filter chain called: `OAuth2AuthorizationRequestRedirectFilter` & `OAuth2LoginAuthenticationFilter`

You can find the applied filters in the console:

> You have to add `@EnableWebSecurity(debug = true) ` in the security configuration, otherwise you won't see the log.

```wiki
 INFO 21493 --- [           main] o.s.s.web.DefaultSecurityFilterChain     : Will secure any request with [
     org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@71b6172c,
     org.springframework.security.web.context.SecurityContextPersistenceFilter@342e690b,
     org.springframework.security.web.header.HeaderWriterFilter@57ddd45b,
     org.springframework.security.web.csrf.CsrfFilter@388d14e,
     org.springframework.security.web.authentication.logout.LogoutFilter@4116f66a,
     org.springframework.security.oauth2.client.web.OAuth2AuthorizationRequestRedirectFilter@6ccac6f4,
     org.springframework.security.oauth2.client.web.OAuth2LoginAuthenticationFilter@438c9aa7,
     org.springframework.security.web.authentication.ui.DefaultLoginPageGeneratingFilter@4e224df5,
     org.springframework.security.web.authentication.ui.DefaultLogoutPageGeneratingFilter@58aa10f4,
     org.springframework.security.web.savedrequest.RequestCacheAwareFilter@78ec89a6,
     org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@5d3b58ca,
     org.springframework.security.web.authentication.AnonymousAuthenticationFilter@4fb56bea,
     org.springframework.security.web.session.SessionManagementFilter@2fb25f4c,
     org.springframework.security.web.access.ExceptionTranslationFilter@16a2ed51,
     org.springframework.security.web.access.intercept.FilterSecurityInterceptor@516592b1
 ]
```

When user clicks the link `/oauth2/authorization/google`, `OAuth2AuthorizationRequestRedirectFilter` will be applied, because this filter contains logic for the base URI (`/oauth2/authorization/`):

```java
package org.springframework.security.oauth2.client.web;

public class OAuth2AuthorizationRequestRedirectFilter extends OncePerRequestFilter {

	/**
	 * The default base {@code URI} used for authorization requests.
	 */
	public static final String DEFAULT_AUTHORIZATION_REQUEST_BASE_URI = "/oauth2/authorization";

    @Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {
		try {
			OAuth2AuthorizationRequest authorizationRequest = this.authorizationRequestResolver.resolve(request);
			if (authorizationRequest != null) {
				this.sendRedirectForAuthorization(request, response, authorizationRequest);
				return;
			}
		}
        // ...
		try {
            // delegates request to the next chain which is OAuth2LoginAuthenticationFilter
			filterChain.doFilter(request, response);
		}
		// ...
	}
}
```

On the `doFilterInternal` method if `authorizationRequest` is not null then, Spring security will redirect to the request **Sign in with Google** page.

But still one question remains unanswered where the `/google` part comes from ? The last part (`/google`) is equal to registration id. And I have setup the registration-id while creating the `ClientRegistration` object.

## Registration Id

In short, registration-id will be:

```java
@Configuration
@EnableWebSecurity(debug = true) // do not debug in the production side
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
   // ...
    private ClientRegistration googleClientRegistration() {
        return  CommonOAuth2Provider.GOOGLE
                .getBuilder("google") // registration-id, after all link will be /oauth2/authorization/google
                .clientId(CLIENT_ID)
                .clientSecret(CLIENT_SECRET)
                .build();
    }
}
```

Spring Security needs to separate different client registrations. To differentiate the client registration it uses the registration-id field. Therefore we should specify the registration-id when we are creating it.

## Summarize what happens when I click `/oauth2/authorization/google`

- Because we added specific filters for the oauth2 with this line `http.oauth2Login();` , Requests with the URI `/oauth2/authorization` will be intercepted by `OAuth2AuthorizationRequestRedirectFilter`
- `OAuth2AuthorizationRequestRedirectFilter` will apply its logic.
- According to the `OAuth2AuthorizationRequestRedirectFilter` logic, user will be redirected to this endpoint:

```wiki
https://accounts.google.com/o/oauth2/v2/auth?
	response_type=code&
	client_id=your_client_id&
	scope=openid%20profile%20email&
	state=xXlYfTiLXz-I9wKD6tNnRGpze...&
	redirect_uri=http://localhost:8080/login/oauth2/code/google&
	nonce=WLa7xnsGF_FZVmDONtHxmQIHS3m6puCwhKZePx3rB50
```

> If you don't know what are the **response_type, client_id, scope etc..**, please read the [https://mehmetozanguven.github.io/spring/2021/09/15/spring-with-oauth2-theory.html](https://mehmetozanguven.github.io/spring/2021/09/15/spring-with-oauth2-theory.html)

- If everything is okey, (I mean, if the client_id, redirect_url and other parameters comply with the Google AuthServer), then user will see the **Sign In with Google Page**
- After successful login with Google, user will be redirected to the http://localhost:8080/login/oauth2/code/google

**Now we have another question, where does the `/login/oauth2/code/google` comes from?**

## Reason for `/login/oauth2/google`

> Don't confuse yourself with the `/login/oauth2/google` and `/oauth2/authorization/google`. These are different URIs for different purposes.

As I said previously, Spring Security decides that whether it redirect the request or not via checking the `OAuth2AuthorizationRequest`. If this object is not null, then it will redirect to the request appropriate AuthServer:

```java
public class OAuth2AuthorizationRequestRedirectFilter extends OncePerRequestFilter {
    // ...
    @Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {
		try {
			OAuth2AuthorizationRequest authorizationRequest = this.authorizationRequestResolver.resolve(request);
			if (authorizationRequest != null) {
				this.sendRedirectForAuthorization(request, response, authorizationRequest);
				return;
			}
		}
    }
    // ...
}
```

While creating the `OAuth2AuthorizationRequest`, Spring Security expects the Authorization Server's redirect URL in the following form:

```wiki
"{baseUrl}/{action}/oauth2/code/{registrationId}"
```

- **baseUrl** : will be the running location of the spring boot application in our case it will be http://localhost:8080
- **action**: If `request.getParameter("action");` is null, then default action will be applied which is **login**
- **registrationId**: is the ClientRegistration.registrationId

**That's was the reason why we set redirect URL: `http://localhost:8080/login/oauth2/code/google` in the google console.**

## What happens after successful login?

Let's assume you allow the client to use your Google accounts. Here are the steps:

> `OAuth2LoginAuthenticationFilter` plays a role in this part.

After successful login with Google, user will be redirected to the http://localhost:8080/login/oauth2/code/google

In the response there will be request's parameter called **`code`** and spring security will use the `request.parameter(code)` value to get access token.

In this time, request won't be intercepted by the `OAuth2AuthorizationRequestRedirectFilter` because request URI (`/login/oauth2/code/google`) does not match with the base URI :

```java
public class OAuth2AuthorizationRequestRedirectFilter extends OncePerRequestFilter {

	/**
	 * The default base {@code URI} used for authorization requests.
	 */
	public static final String DEFAULT_AUTHORIZATION_REQUEST_BASE_URI = "/oauth2/authorization";
}
```

If request won't be intercepted by the `OAuth2AuthorizationRequestRedirectFilter`, then there will be no redirection to the AuthServer. Request will be delegated to the next chain called **`OAuth2LoginAuthenticationFilter`**

If the request needs to be authenticated (in our case answer is the yes), then `OAuth2LoginAuthenticationFilter` runs the `attemptAuthentication` method.

> Can you remember the steps to authenticate request(s) in the spring security (from my previous blog). For instance, when your custom filter needs to authenticate any request(s), it should follow these steps (as Spring Security team suggests) :
>
> - Create a custom filter
> - Autowired the appropriate `AuthenticationManager`
> - Create the `CustomAuthenticationToken` for your custom filter
> - In the filter, call the `authenticationManager.authenticate(customAuthenticationToken);`
> - `AuthenticationManager` will try to find correct `AuthenticationProvider` and call the `authenticationProvider.authenticate(customAuthenticationToken)`
> - `AuthenticationProvider` will return fully authenticated `customAuthenticationToken` or throw an error
>
> If you don't remember or don't know, please refer to the [https://mehmetozanguven.github.io/spring/2020/12/30/spring-security-5-custom-filter.html](https://mehmetozanguven.github.io/spring/2020/12/30/spring-security-5-custom-filter.html)
>
> This same login will be applied by the `OAuth2LoginAuthenticationFilter`

`OAuth2LoginAuthenticationFilter` will create the `OAuth2AuthenticationToken` and send this object to the various Authentication Providers to get fully authenticated object.

One of the `AuthenticationProvider` will send the **`code`** from the request and get the accessToken from Google.

After getting the accessToken, Spring Security will send **accessToken** to the Resource Server(in our case it is Google again) get the user's account information (such as name, email etc..)

After getting user's account information, Spring Security will create a fully authenticated object with the type `OAuth2AuthenticationToken`.

Because Filter now has the fully authenticated object, it can run the `successfulAuthentication` method. (Default behaviour for successful authentication):

```java
protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain,
  Authentication authResult) throws IOException, ServletException {
  SecurityContextHolder.getContext().setAuthentication(authResult); // 1
  if (this.logger.isDebugEnabled()) {
    this.logger.debug(LogMessage.format("Set SecurityContextHolder to %s", authResult));
  }
  this.rememberMeServices.loginSuccess(request, response, authResult);
  if (this.eventPublisher != null) {
    this.eventPublisher.publishEvent(new InteractiveAuthenticationSuccessEvent(authResult, this.getClass()));
  }
  this.successHandler.onAuthenticationSuccess(request, response, authResult); // 2
}
```

1.  Fully authenticated object will be stored in the `SecurityContext`
2.  Because we don't have custom implementation of the `AuthenticationSuccessHandler` interface, Spring Security will redirect to the user `http://localhost:8080/`

I hope, it is clear right now.

## Conclusion

- In this blog, we go through the details of the filter specific to the OAuth2 process. Basically there are 2 filters for the OAuth2:

  - **`OAuth2AuthorizationRequestRedirectFilter`** : Responsible for redirection (to the endpoint of the Authorization Server)
  - **`OAuth2LoginAuthenticationFilter`**: Responsible to authenticate OAuth2 request(s)

- `/oauth2/authorization/{registrationId}` is the endpoint which redirection to the endpoint of the Authorization Server will be applied. (Redirection will be applied by the `OAuth2AuthorizationRequestRedirectFilter` )

- Spring security expects Authorization Server redirect URI in this format: (therefore when you set the redirect URIs in the console.cloud.google or other providers such as Github, Facebook, you should consider it)

```wiki
"{baseUrl}/{action}/oauth2/code/{registrationId}"
```

- Default action is **login**
- baseURL is the running location of the spring boot (for your local environment, it will be http://localhost:8080)
- registrationId is the one, you specified when you create `ClientRegistration`

Last but not least, wait for the next one ...
