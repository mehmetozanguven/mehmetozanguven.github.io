---
layout: post
title: Spring Security -- 8) Spring Security CSRF Attack Simulation & CSRF Setup and Customization (Cross-Site Request Forgery)
date: 2021-05-16 22:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

In this post, we are going to learn **what CSRF(or XSRF) attack is**, **how to simulate csrf attack in spring** and **how to setup csrf protection in spring application**

Topics are:

- [**Github Link**](#github_link)
- [**What is the CSRF attack?**](#csrf_attacks)
  - [**How Does CSRF attacks work?**](#how_csrf_works)
  - [**CSRF Protection**](#csrf_protect)
- [**Project Setup**](#project_Setup)
  - [**Where is the CSRF token**](#where_is_the_csrf_token)
  - [**Create PasswordChange Post Endpoint**](#post_endpoint)
- [**Disable CSRF protection in Spring Boot**](#disable_csrf_protection)
- [**CSRF attack simulation**](#attack_simulation)
  - [**Attacker creates a simple html page**](#create_html_page)
  - [**Simulate attack**](#simulate_attack)
- [**Protect page with random csrf token**](#protect_page)
  - [**Protect change password form**](#protect_password_form)
  - [**Simulate attack with CSRF protection**](#attack_simulation_with_csrf)
- [**How Random Token Generated**](#how_generate_token)
- [**Where to store CSRF token**](#where_to_store_token)
- [**How to debug CSRF operations**](#how_to_debug)
- [**How to customize CSRF token**](#how_to_customize)
  - [**How to disable CSRF protection for specific endpoint(s)**](#disable_csrf_for_endpoint)
  - [**How to customize CSRF repository**](#change_token_algorithm)

## Github Link <a name="github_link"></a>

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/spring-security-examples/tree/master/spring-security-csrf-setup)

## What is the CSRF(Cross site request forgery) attack <a name="csrf_attacks"></a>

- Cross-site request forgery is a web security vulnerability which an attacker can trick a user into clicking a malicious link that triggers undesirable or unexpected side effects.
- This attack allows an attacker to induce users to perform actions that they do not intend to perform.
- In a successful CSRF attack might change user email/password address on the system.
- Usually GET requests is used by an attacker. Because GET requests are the only type of HTTP request that contains the entire of the request's contents in a URL.

### How Does CSRF attacks work? <a name="how_csrf_works"></a>

- Three conditions must match:

  1. There must be an action such that modifying permission for users (for example, changing password
     action, email action etc..)
  2. Action should be performed on cookie based session. For example if one user wants to change his/her
     password, he/she should send his/her cookie to the server. (otherwise server can not know the correspond user)
  3. There will be none unpredictable request parameters. For example when user is sending post request, creating & sending random request parameter(this can be done in the server side) can be a problem for attacker.

- Let's say there is post request to change email address of user, like this one:

```wiki
POST /email/change HTTP/1.1
Host: sample.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 30
Cookie: session=abcsdf123dacxvç
email=user@gmail.com
// This request is suitable for CSRF attacks, because action is simple and modifying
permission, session cookie is used to recognize user, and there is no unpredictable
post parameter
```

- Right now, attacker can create a web site like this one:

```html
<html>
  <body>
    <form action="https://sample.com/email/change" method="POST">
      <input type="hidden" name="email" value="attackEmail@attack.net" />
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```

- If a victim user visits the attacker's webpage, the following will happen:
  - Attacker's page will trigger HTTP post request to the vulnerable website.
  - If user is logged in to the sample.com, user's browser will automatically include user's session cookie in
    the request.
  - Then attacker's webpage will process the request in the normal way, after all attacker will change the
    user's email with something else.

### CSRF Protection <a name="csrf_protect" ></a>

- To protect from CSRF attacks, do not allow GET request to change the state of the server. Website should
  use GET requests only to fetch web pages or other resources.
- POST requests may do some sensitive stuff. Only our forms and JavaScript must perform these actions. To
  do that we use randomized anti-CSRF cookie

> Cookies are small pieces of text passed back and forth between the browser and web server in HTTP
> headers. If the web server returns an HTTP response containing a header value like Set-Cookie:
> \_xsrf=5978e29d4ef434a1 , the browser will send back the same information in the next HTTP request
> in a header with form Cookie: \_xsrf=5978e29d4ef434a1

- Secure websites use anti-CSRF cookies to verify that POST requests originate from pages hosted on the
  same web domain.
- However using cookies are not the only solution for CSRF attack, because an attacker can steal csrf token
  and use it for subsequent requests. **Therefore we need a solution such as only send the cookie which was
  initiated by the webserver. Solution for that is SameSite=Strict**

```wiki
Set-Cookie: _xsrf=5978e29d4ef434a1; SameSite=Strict;
```

- With this instruct, browser will only send the cookies initiated by web-server not third-party

## Project Setup <a name="project_Setup"></a>

To simulate CSRF attacks, first create a simple spring boot project. This project will contain the following dependencies:

- Spring Web
- Spring Security
- Thymeleaf

And create home controller which returns the homePage.html:

```java
@Controller
public class HomeController {

    @GetMapping
    public String homePage() {
        return "homePage";
    }
}
```

Create `homePage.html` inside the `resources/templates` folder:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Title</title>
  </head>
  <body>
    <p>HomePage</p>
  </body>
</html>
```

Run the project, and try to connect `localhost:8080` , you will be redirected to the login page, username will be **user** and password can be found in the console:

```wiki
// ...
Using generated security password: fa146166-c0ce-4e85-add0-e8dc48daaf00
// ...
```

After successful login, you will see the homePage.

To logout just go to the `http://localhost:8080/logout`

### Where is the CSRF token <a name="where_is_the_csrf_token"></a>

Spring Security will enable the CSRF token by default, if you want to see where csrf token is, after logout inspect the login form and you will see the hidden variable called `_csrf` :

<img src="/assets/spring/security/csrf/default_csrf.png" alt="default_csrf.png" />

### Create PasswordChange Post Endpoint <a name="post_endpoint" />

First create a post endpoint (just assume that this post endpoint is responsible to change user's password):

```java
@Controller
public class PasswordController {
    private final CustomerPasswordService customerPasswordService;

    public PasswordController(@Autowired CustomerPasswordService customerPasswordService) {
        this.customerPasswordService = customerPasswordService;
    }

    @PostMapping("/changePassword")
    public String changeCustomerPassword(@RequestParam String newPassword) {
        System.out.println("New Password will be: " + newPassword);
        customerPasswordService.changePassword(newPassword);
        model.addAttribute("newPassword", newPassword);
        return "passwordChanged";
    }
}
```

`CustomerPasswordService` is just a dummy service:

```java
@Service
public class CustomerPasswordService {

    public void changePassword(String newPassword) {
        // dummy method
        System.out.println("Customer password was changed to: " + newPassword);
    }
}
```

Update the `homePage.html` (just add the form )

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Title</title>
  </head>
  <body>
    <p>HomePage</p>
    <form method="post" action="/changePassword">
      <p>Write your new password:</p>
      <input type="password" name="newPassword" />
      <button type="submit">Change My Password</button>
    </form>
  </body>
</html>
```

Create the `passwordChanged.html` :

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Title</title>
  </head>
  <body>
    <h2>
      Your password was changed. New Password is :
      <span th:text="${newPassword}"></span>
    </h2>
  </body>
</html>
```

Run the application and press the button, you will be redirected to the passwordChanged.html and you will see your new password also in the console you will see the following outputs:

```wiki
New Password will be: asd
Customer password was changed to: asd
```

## Disable CSRF protection <a name="disable_csrf_protection"></a>

To disable CSRF protection, just override configuration `configure(HttpSecurity http)`

> **Disabling the CSRF protection is not recommended**

```java
@Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter  {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http);
        http.csrf().disable();
    }
}
```

## CSRF attack simulation <a name="attack_simulation"/>

> Make sure that you disabled the CSRF protection

The best way to understand why CSRF protection is needed is to generate attack and to see what's happening in the background.

### Attacker creates a simple html page <a name="create_html_page" />

Create an simple html file called `attacker.html`

Let's assume that I am a influencer and somebody wants to connect me :) (because I have done awesome jobs/works etc.. according to the attacker :) )

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Fake Email from attacker</title>
  </head>
  <body>
    <h1>Can I get your feedback on career advice?</h1>
    <form method="post" action="http://localhost:8080/changePassword">
      <p>
        Hey Mehmet Ozan Guven, I hope you're doing well! I'm reaching out
        because I'm thinking of making a career change and would love to get
        your advice. I admire you and all you've accomplished. I would be eager
        to have your input. Free for coffee sometime this week?
      </p>
      <input type="hidden" name="newPassword" value="attackNewPassword" />
      <button type="submit">See Contact Information</button>
    </form>
  </body>
</html>
```

Please look at the action, it is the our changePassword endpoint (which of course full url) and also attacker will try to set the legitimate customer's password something else.

### Simulate Attack <a name="simulate_attack" />

ow, when you are logged-in the application, please open the `attacker.html` page in the browser chrome/firefox etc.. then click the button **See Contact Information**

After clicking the button, you will be redirected to the page `http://localhost:8080/changePassword` with this page:

```html
Your password was changed. New Password: attackNewPassword
```

You can verify that in the console as well:

```wiki
New Password will be: attackNewPassword
Customer password was changed to: attackNewPassword
```

**That's the CSRF attack, attacker triggers the user to perform an action on behalf of the user**

Now, let's see how spring security protect our customer(s).

## Protect page with random csrf token <a name="protect_page" />

> Enable the csrf protection via comment out this line:
>
> ```java
> @Configuration
> public class SecurityConfiguration extends WebSecurityConfigurerAdapter  {
>
>     @Override
>     protected void configure(HttpSecurity http) throws Exception {
>         super.configure(http);
> //        http.csrf().disable(); // not recommended
>     }
> }
> ```
>
> and re-run the project:

When we enable csrf protection:

- Spring boot will generate random token (hard to guess for attacker)
- When performing mutation actions (such as post, put, delete requests), spring security will look for the token, if token was not found, spring security will reject the request)

After successful login, try to change your password in the homePage. You will get the **Whitelabel Error Page** response, after submitting the form:

```html
Whitelabel Error Page This application has no explicit mapping for /error, so
you are seeing this as a fallback. Wed May 12 16:57:38 TRT 2021 There was an
unexpected error (type=Forbidden, status=403).
```

You can also verify that in the console (There will be no outputs)

> ### How Did I Pass the Login Form?
>
> You may ask "if I can not pass the password change form, how did i pass the login form?" answer is: Spring will automatically add the csrf token in the request (take look at the picture in the section **Where is the CSRF token ?**)

Let's protect the password form.

### Protect change password form <a name="protect_password_form" />

With thymeleaf engine, it is easy: (update the `homePage.html`)

```html
<!DOCTYPE html>
<html lang="en">
  <html xmlns:th="https://www.thymeleaf.org">
    <head>
      <meta charset="UTF-8" />
      <title>Title</title>
    </head>
    <body>
      <p>HomePage</p>
      <form method="post" action="/changePassword">
        <p>Write your new password:</p>
        <input type="password" name="newPassword" />
        <input
          type="hidden"
          th:name="${_csrf.parameterName}"
          th:value="${_csrf.token}"
        />
        <button type="submit">Change My Password</button>
      </form>
    </body>
  </html>
</html>
```

Run the project. In the password form, inspect the elements:

<img src="/assets/spring/security/csrf/passwordFormCsrf.png" alt="passwordFormCsrf.png" />

Now, we have the csrf token, customer can change his/her password

Now, try to change customer password with attacker email

### Simulate attack with CSRF protection <a name="attack_simulation_with_csrf" />

Just run the attack email in browser (when you are logged-in) and click the link. You will get the **Whitelabel error page**, because there is not csrf token then Spring security will reject the request.

## How Random Token Generated <a name="how_generate_token" />

- Just look at the `HttpSessionCsrfTokenRepository`, you will see this method:

```java
private String createNewToken() {
		return UUID.randomUUID().toString();
}
```

## Where to store CSRF token <a name="where_to_store_token" />

By default csrf token stored in the HttpSession and validated by server-side. In spring security `HttpSessionCsrfTokenRepository` is responsible for that.

## How to debug CSRF operations <a name="how_to_debug" />

Just add the debug point to the `CsrfFilter.doFilterInternal(...)` method

## How to customize CSRF token <a name="how_to_customize" />

You can customize with `CsrfConfigurer<HttpSecurity>`

### How to disable CSRF protection for specific endpoint(s) <a name="disable_csrf_for_endpoint" />

Here is the example configuration for that, you can use it for your project:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter  {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http);
//        http.csrf().disable(); // not recommended
        http.csrf(c -> {
           c.ignoringAntMatchers("/disabledEndpoint", "/anotherEndpointParent/**");
        });
    }
}
```

### How to customize CSRF repository algorithm <a name="change_token_algorithm" />

Spring Security will provide an interface called `interface CsrfTokenRepository` to customize everything related to CSRF.

```java
public interface CsrfTokenRepository {
	CsrfToken generateToken(HttpServletRequest request);
	void saveToken(CsrfToken token, HttpServletRequest request,
			HttpServletResponse response);
	CsrfToken loadToken(HttpServletRequest request);
}

```

`CsrfToken` is the another interface to provide information about CSRF token in the spring application:

```java
public interface CsrfToken extends Serializable {
	String getHeaderName();
	String getParameterName();
	String getToken();
}
```

You can implement your own repository.

> Do not customize the csrf repository unless you know what you are doing. Custom implementation can cause security vulnerabilities !!

You can find the example in my github repo , especially: `MyCsrfRepository`

And update configuration:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter  {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http);
//        http.csrf().disable(); // not recommended
        http.csrf(c -> {
           c.ignoringAntMatchers("/disabledEndpoint", "/anotherEndpointParent/**");
           c.csrfTokenRepository(new MyCsrfRepository());
        });
    }
}
```

I will continue with the next one
