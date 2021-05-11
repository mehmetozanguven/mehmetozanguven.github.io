---
layout: post
title: Spring Security -- 6) Multiple Authentication Filters && Providers
date: 2021-01-06 23:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

In this post, let's implement two steps authentication mechanism. This will be similar to JWT authentication but instead of JWT I will use my implementation. After legitimate user makes the login request, my response will include the header value called **Authorization** and user can be able to access to restricted endpoints using the Authorization header's value.

> You may see the previous [post](https://mehmetozanguven.github.io/spring/2020/12/30/spring-security-5-custom-filter.html) which includes simple project to understand filter, provider and authentication.

> By the way, if you like the my blog contents, you can support me via [patreon](https://www.patreon.com/mehmetozanguven) :)

Topics are:

- [**Github Link**](#github_link)
- [**Project Overview**](#project_overview)
- [**Default Project Setup**](#default_project_setup)
  - [Database Setup](#database_setup)
  - [Password Encoder](#password_encoder)
  - [Entities, Repositories, Controller and Config Classes](#default_classes)
- [**Creating Filter and Providers**](#create_filter_and_providers)
  - [UsernamePasswordAuthFilter](#usernamepassword_auth_filter)
  - [Authentication Instances](#authentication_instances)
  - [AuthFilter passes Authentication instances to AuthManager](#auth_filter_to_auth_manager)
  - [Creating Provider(s)](#creating_provider)
    - [UsernamePasswordAuthProvider](#usernamepassword_auth_provider)
    - [OtpAuthProvider](#otp_auth_provider)
    - [Add the Providers to the configuration](#provider_to_configuration)
  - [Add the Filter to the configuration](#filter_to_configuration)
- [**Creating the second filter**](#second_filter)
  - [Second filter Authentication Instance](#second_authentication_instances)
  - [Second filter Authentication Provider](#second_filter_provider)
  - [Add the second filter and provider to the configuration](#add_second_to_configuration)

## Github Link <a name="github_link"></a>

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/spring-security-examples/tree/master/spring-security-multiple-providers)

## Project Overview <a name="project_overview"></a>

This project will be kind of a JWT token authentication. First, User will be authenticated against the database via **username and password**. After database authentication, I will generate one-time-password(otp) for that user which is actually random code. After that user will send request which also includes this new random otp. Then I will check the otp value, if the otp value is correct, then I will generate Authorization header's value and save it to my database **(in this simple project database will be the `HashSet`, don't do this in real case scenario(s))** and my response will include **Authorization: {{Authorization header's value }}** header which is the random token to access the restricted endpoint(s).

For other requests than the `/login`, I will get the Authorization header's value and check the whether the value is correct or not. If it is not correct, authentication will fail.

> This might not be considered as secure application. There could be situation where access token can be stolen!! For instance, otp value in the project has no expiration time, but in real case it should be...

After starting the project run the following command (I assume that there is user with test_user and password=1234 in your `users` table )

If you try the connect the restricted endpoint without authorization, the you will get an exception:

```bash
$ curl  -X GET http://localhost:8080/hello

// log in the console:
org.springframework.security.authentication.BadCredentialsException: Authorization value is not correct
	at com.mehmetozanguven.springsecuritymultipleproviders.service.providers.TokenAuthProvider.authenticate(TokenAuthProvider.java:27) ~[classes/:na]
```

Before getting the authorization token, you should create an record into the otp table:

```bash
$ curl -H "username:test_user" -H "password:1234" -X GET http://localhost:8080/login
```

Right now look at the otp table, you should have one record:

```bash
testdatabase=# select * from otp;
 id | username  |    otp
----+-----------+------------
  1 | test_user | RXyObQYNDr
```

Right now, run the following curl command:

```bash
$ curl -H "username:test_user" -H "otp:RXyObQYNDr" -X GET http://localhost:8080/login -v
Note: Unnecessary use of -X or --request, GET is already inferred.
*   Trying ::1:8080...
* Connected to localhost (::1) port 8080 (#0)
> GET /login HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.69.1
> Accept: */*
> username:test_user
> otp:RXyObQYNDr
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200
-------
< Authorization: 7c60f6b3-4047-4fc6-8a37-267661c574f4
-------
< X-Content-Type-Options: nosniff
< X-XSS-Protection: 1; mode=block
< Cache-Control: no-cache, no-store, max-age=0, must-revalidate
< Pragma: no-cache
< Expires: 0
< X-Frame-Options: DENY
< Content-Length: 0
< Date: Mon, 04 Jan 2021 18:45:49 GMT
<
* Connection #0 to host localhost left intact

```

**Authorization: 7c60f6b3-4047-4fc6-8a37-267661c574f4** is the token to access the restricted endpoint

Now access the restricted endpoint via Authorization value:

```bash
$ curl -H "Authorization:7c60f6b3-4047-4fc6-8a37-267661c574f4" -X GET http://localhost:8080/hello
hello
```

If you try to access with the wrong value:

```bash
$ curl -H "Authorization:test_value" -X GET http://localhost:8080/hello

// log in the console:
org.springframework.security.authentication.BadCredentialsException: Authorization value is not correct
	at com.mehmetozanguven.springsecuritymultipleproviders.service.providers.TokenAuthProvider.authenticate(TokenAuthProvider.java:27) ~[classes/:na]
```

## Default Project Setup <a name="default_project_setup"></a>

In this simple project, you will need an JPA and database connection setup (I am going to use Postgresql):

```xml
<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.postgresql</groupId>
			<artifactId>postgresql</artifactId>
			<scope>runtime</scope>
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

### Database Setup <a name="database_setup"></a>

Do not forget to add connection setup to database, here is my configuration:

```properties
# Spring DATASOURCE for local postgresql
spring.datasource.url=jdbc:postgresql://localhost:5432/testdatabase
spring.datasource.username=postgres
spring.datasource.password=1234

# For localdb, it would be good to re-create all-tables
spring.jpa.hibernate.ddl-auto=update
```

And please setup empty table called `users` which has 3 columns:

```bash
[mehmetozanguven@localhost ~]$ sudo -iu postgres
[postgres@localhost ~]$ psql
psql (12.5)
Type "help" for help.

postgres=# \c testdatabase
You are now connected to database "testdatabase" as user "postgres".
testdatabase=# drop table users ; // drop the table if you have previously...
DROP TABLE
testdatabase=# CREATE TABLE IF NOT EXISTS users (id serial PRIMARY KEY, username VARCHAR(50), password VARCHAR(50));
CREATE TABLE
```

You should also setup another table called `otp` :

```bash
testdatabase=# CREATE TABLE IF NOT EXISTS otp (id serial PRIMARY KEY, username VARCHAR(50), otp VARCHAR(50));
CREATE TABLE
```

Do not forget to add test user:

```bash
testdatabase=# INSERT INTO users (username, password) VALUES ('test_user', '1234');
INSERT 0 1
testdatabase=# select * from users;
 id | username  | password
----+-----------+----------
  1 | test_user | 1234
(1 row)
```

### Password Encoder <a name="password_encoder"></a>

I will use `NoopPasswordEncoder` for this simple project.

### Entities, Repositories, Controller and Config Classes <a name="default_classes"></a>

Controller:

```java
package com.mehmetozanguven.springsecuritymultipleproviders.controllers;

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

Entities:

```java
package com.mehmetozanguven.springsecuritymultipleproviders.entities;

import javax.persistence.*;

@Entity
@Table(name = "users")
public class UserDTO {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column(name = "username")
    private String username;

    @Column(name = "password")
    private String password;
}
```

```java
package com.mehmetozanguven.springsecuritymultipleproviders.entities;

import javax.persistence.*;

@Entity
@Table(name = "otp")
public class OtpDTO {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column(name = "username")
    private String username;

    @Column(name = "otp")
    private String otp;
}
```

Repositories:

```java
package com.mehmetozanguven.springsecuritymultipleproviders.repositories;

import com.mehmetozanguven.springsecuritymultipleproviders.entities.UserDTO;
import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository extends JpaRepository<UserDTO, Long> {
     Optional<UserDTO> findUserDTOByUsername(String username);
}
```

```java
package com.mehmetozanguven.springsecuritymultipleproviders.repositories;

import com.mehmetozanguven.springsecuritymultipleproviders.entities.OtpDTO;
import org.springframework.data.jpa.repository.JpaRepository;

public interface OtpRepository extends JpaRepository<OtpDTO, Long> {
}
```

Model to convert `UserDTO` to `UserDetails` object:

```java
public class SecureUser implements UserDetails {

    private final UserDTO userDTO;

    public SecureUser(UserDTO userDTO) {
        this.userDTO = userDTO;
    }

    /**
     * Because I will not look at the authority part for now
     * I am just creating dummy authority for the users
     * @return
     */
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(() -> "read");
    }

    @Override
    public String getPassword() {
        return userDTO.getPassword();
    }

    @Override
    public String getUsername() {
        return userDTO.getUsername();
    }
    // ...
}
```

UserDetailsService:

```java
package com.mehmetozanguven.springsecuritymultipleproviders.service;

@Service
public class PostgresqlUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        UserDTO userInDb = userRepository.findUserDTOByUsername(username)
                                .orElseThrow(() -> new UsernameNotFoundException("User not found in the db"));
        return new SecureUser(userInDb);
    }
}
```

Configuration:

```java
@Configuration
public class ProjectBeanConfiguration extends WebSecurityConfigurerAdapter {

    @Bean
    public PasswordEncoder passwordEncoder(){
        return NoOpPasswordEncoder.getInstance();
    }
}
```

## Creating Filter and Providers <a name="create_filter_and_providers"></a>

I do not want to convert `ServletRequest` to the `HttpServletRequest` instead I will directly work on the `HttpServletRequest` extending `OncePerRequestFilter`

And also I am going to override method `OncePerRequestFilter#shouldNotFilter(HttpServletRequest request)` to determine for which path(s) this filter will be called.

### UsernamePasswordAuthFilter <a name="usernamepassword_auth_filter"></a>

This filter will be called when path is `/login`

```java
public class UsernamePasswordAuthFilter extends OncePerRequestFilter {

    @Autowired
    // but I didn't defined any manager YET !!!
    // let's define the one inside the configuration
    private AuthenticationManager authenticationManager;

    @Override
    protected void doFilterInternal(HttpServletRequest httpServletRequest,
                                    HttpServletResponse httpServletResponse,
                                    FilterChain filterChain) throws ServletException, IOException {
        String username = httpServletRequest.getHeader("username");
        String password = httpServletRequest.getHeader("password");
        String otp = httpServletRequest.getHeader("otp");

        if (otp == null){
            // authenticate via username and password and generate one time password
        }else{
            // authenticate via one time password and
            // generate Authorization header with random value
            // finally save the header's value elsewhere!!
        }

    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) throws ServletException {
        if (isPathLogin(request)){
            // if path is login then return false, which means: call the doFilterInternal
            return false;
        }else{
            // do not call doFilterInternal
            return true;
        }
    }

    private boolean isPathLogin(HttpServletRequest request){
        return request.getServletPath().equals("/login");
    }
}
```

Before diving into authentication instances, first let's define one manager in the configuration class:

```java
@Configuration
public class ProjectBeanConfiguration extends WebSecurityConfigurerAdapter {
	// ...

    @Override
    @Bean
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
}
```

As you recall from the previous [post](https://mehmetozanguven.github.io/spring/2020/12/30/spring-security-5-custom-filter.html#create_auth_instance) , there could be `Authentication` instance and `AuthenticationManager` will call the appropriate `AuthenticationProvider` based on the type of the Authentication instance(s). Thus, I should create Authentication classes for each authentication logics.

### Authentication Instances <a name="authentication_instances"></a>

For simplicity, I will extend the `UsernamePasswordAuthenticationToken` . Here is the Authentication class for otp authentication:

```java
package com.mehmetozanguven.springsecuritymultipleproviders.service.authentications;

public class OtpAuthentication extends UsernamePasswordAuthenticationToken {
    // this constructor creates a Authentication instance which is not fully authenticated
    public OtpAuthentication(Object principal, Object credentials) {
        super(principal, credentials);
    }

    // this constructor creates a Authentication instance which is fully authenticated
    public OtpAuthentication(Object principal, Object credentials, Collection<? extends GrantedAuthority> authorities) {
        super(principal, credentials, authorities);
    }
}
```

For Username:password authentication, I will use `UsernamePasswordAuthentication`

```java
package com.mehmetozanguven.springsecuritymultipleproviders.service.authentications;

public class UsernamePasswordAuthentication extends UsernamePasswordAuthenticationToken {
    // this constructor creates a Authentication instance which is not fully authenticated
    public UsernamePasswordAuthentication(Object principal, Object credentials) {
        super(principal, credentials);
    }

    // this constructor creates a Authentication instance which is fully authenticated
    public UsernamePasswordAuthentication(Object principal, Object credentials, Collection<? extends GrantedAuthority> authorities) {
        super(principal, credentials, authorities);
    }
}
```

After my Authentication instances are ready, Filter is ready to pass the instances to the AuthenticationManager

### AuthFilter passes Authentication instances to AuthManager <a name="auth_filter_to_auth_manager"></a>

After all Filter will get the result from the Manager.

```java
public class UsernamePasswordAuthFilter extends OncePerRequestFilter {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Override
    protected void doFilterInternal(HttpServletRequest httpServletRequest,
                                    HttpServletResponse httpServletResponse,
                                    FilterChain filterChain) throws ServletException, IOException {
        String username = httpServletRequest.getHeader("username");
        String password = httpServletRequest.getHeader("password");
        String otp = httpServletRequest.getHeader("otp");

        if (otp == null){
            // * if there is no one-time-password, then generate an OTP
            // 1) first make sure that, legitimate user is connecting to the your system
            UsernamePasswordAuthentication usernamePasswordAuthentication =
                    new UsernamePasswordAuthentication(username, password);
            // 2) authenticationManager will find the correct provider for that authentication
            Authentication resultForUsernamePassword = authenticationManager.authenticate(usernamePasswordAuthentication);
            // 3) Because authenticationManager.authenticate will return fully authenticated instance
            // ( otherwise it must throw an error), You can generate otp for authenticatied user
            // 4) Generate new otp
            String otpCode = RandomStringUtils.randomAlphabetic(10);
            // 5) save this new one-time-password for that user.
            OtpDTO otpDTO = new OtpDTO();
            otpDTO.setUsername(username);
            otpDTO.setOtp(otpCode);
            otpRepository.save(otpDTO);
        }else{
            // if there is a one-time-password, authenticate user via one-time-password(otp)
            OtpAuthentication otpAuthentication = new OtpAuthentication(username, otp);
            // authenticationManager will find the correct provider for that authentication
            Authentication resultForOtp = authenticationManager.authenticate(otpAuthentication);
            // after getting fully authenticated instance, generate authorization header' value
            String authValue = UUID.randomUUID().toString();
            // save the authorization value for checking future request(s)
            authorizationTokenHolder.add(authValue);
            // add new header to the response
            httpServletResponse.setHeader("Authorization", authValue);
        }
    }
	// ...
}
```

What I am missing is that, **there should be provider(s) for these authentication processes**. Let's define the providers

### Creating Providers <a name="creating_provider"></a>

Let's recap what the AuthenticationProvider does:

- In general `AuthenticationProvider` contains two methods: `authenticate()` contains the authentication logic and `supports()` contains the logic which this authentication provider should be applied or not.

For the `authenticate()` method, there are three options:

1. If the request is authenticated, should return authenticated Authentication instance
2. If the request is not authenticated, throw AuthenticationException
3. If the Authentication isn’t supported by this provider, then return null, in another words AuthenticationProvider will say the AuthenticationManager: “**Hey AuthenticationManager please try to use another Providers, I am not responsible for this**”

Right now, I need two AuthenticationProvider for each Authentication processes (`OtpAuthentication && UsernamePasswordAuthentication` )

#### UsernamePasswordAuthProvider <a name="usernamepassword_auth_provider"></a>

```java
package com.mehmetozanguven.springsecuritymultipleproviders.service.providers;

@Component
public class UsernamePassswordAuthProvider implements AuthenticationProvider {

    @Autowired
    private PostgresqlUserDetailsService postgresqlUserDetailsService;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String username = authentication.getName();
        String password = (String) authentication.getCredentials();

        // loadByUsername will throw an exception if user is not found
        UserDetails userDetails = postgresqlUserDetailsService.loadUserByUsername(username);

        // after finding the user's details, check with the passwordEncoder
        if (passwordEncoder.matches(password, userDetails.getPassword())){
            // if everything is correct, create a fully authenticated object
            // for this simple project, authorities is hard-coded, do not care about
            return new UsernamePasswordAuthenticationToken(username, password, List.of(() -> "read"));
        }

        throw new BadCredentialsException("BadCredentialException");
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthentication.class.equals(authentication);
    }
}
```

#### OtpAuthProvider <a name="otp_auth_provider"></a>

```java
package com.mehmetozanguven.springsecuritymultipleproviders.service.providers;

@Component
public class OtpAuthProvider implements AuthenticationProvider {

    @Autowired
    private OtpRepository otpRepository;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String username = authentication.getName();
        String otp = (String) authentication.getCredentials();

        OtpDTO otpDTO = otpRepository.findOtpDtoByUsername(username)
                .orElseThrow(() -> new BadCredentialsException("Bad Otp Credentials"));

        return new OtpAuthentication(username, otp, List.of(() -> "read"));
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return OtpAuthentication.class.equals(authentication);
    }
}
```

The left is to add these providers to the configuration

#### Add the Providers to the configuration <a name="provider_to_configuration"></a>

```java
@Configuration
public class ProjectBeanConfiguration extends WebSecurityConfigurerAdapter {
    @Autowired
    private UsernamePassswordAuthProvider usernamePassswordAuthProvider;

    @Autowired
    private OtpAuthProvider otpAuthProvider;

    // ...
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.authenticationProvider(usernamePassswordAuthProvider)
            .authenticationProvider(otpAuthProvider);
    }
}
```

### Add the Filter to the configuration <a name="filter_to_configuration"></a>

```java
@Configuration
public class ProjectBeanConfiguration extends WebSecurityConfigurerAdapter {
    @Autowired
    private UsernamePasswordAuthFilter usernamePasswordAuthFilter;

    // ...

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.addFilterAt(usernamePasswordAuthFilter, BasicAuthenticationFilter.class);
    }
}
```

I can create one-time-password, and using this otp I can also generate Authorization header.
It is time to authenticate (allow users to access restricted enpoints) using this Authorization header.

## Creating the second filter <a name="second_filter"></a>

The second filter will look at the Authorization header in the request and it will allow the request to access endpoint or not.

Because `UsernamePasswordAuthFilter` was responsible to process the authentication for only `/login` endpoint, all other endpoints must be filtered.

```java
package com.mehmetozanguven.springsecuritymultipleproviders.service.filters;

public class TokenAuthFilter extends OncePerRequestFilter {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        String authorization = request.getHeader("Authorization");

        TokenAuthentication tokenAuthentication = new TokenAuthentication(null, authorization);

        Authentication fullyAuthentication = authenticationManager.authenticate(tokenAuthentication);
        SecurityContextHolder.getContext().setAuthentication(fullyAuthentication);
        filterChain.doFilter(request, response);
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) throws ServletException {
        if (isPathLogin(request)){
            // if path is login then return true,
            // which means: do not run the TokenAuthFilter#doFilterInternal,
            // this authentication must be done by UsernamePasswordAuthFilter
            return true;
        }else{
            // for other endpoints run this authentication
            return false;
        }
    }

    private boolean isPathLogin(HttpServletRequest request){
        return request.getServletPath().equals("/login");
    }
}
```

As I have done previously, I should create a provider and authenticate instance for that filter. Provider will include the authentication login.

### Second filter Authentication Instance <a name="second_authentication_instances"></a>

Right now, I will have `TokenAuthentication` which extends `UsernamePasswordAuthenticationToken`

```java
package com.mehmetozanguven.springsecuritymultipleproviders.service.authentications;

import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.GrantedAuthority;

import java.util.Collection;

public class TokenAuthentication extends UsernamePasswordAuthenticationToken {
    // this constructor creates a Authentication instance which is not fully authenticated
    public TokenAuthentication(Object principal, Object credentials) {
        super(principal, credentials);
    }

    // this constructor creates a Authentication instance which is fully authenticated
    public TokenAuthentication(Object principal, Object credentials, Collection<? extends GrantedAuthority> authorities) {
        super(principal, credentials, authorities);
    }
}
```

Now AuthenticationManager will call the correct authentication provider for that second filter. Let's implement the provider

### Second filter Authentication Provider <a name="second_filter_provider"></a>

```java
@Component
public class TokenAuthProvider implements AuthenticationProvider {
    @Autowired
    private AuthorizationTokenHolder authorizationTokenHolder;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String authorizationToken = (String) authentication.getCredentials();
        boolean isCorrectToken = authorizationTokenHolder.contains(authorizationToken);

        if (isCorrectToken){
            return new TokenAuthentication(null, authorizationToken, List.of(() -> "read"));
        }else {
            throw new BadCredentialsException("Authorization value is not correct");
        }
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return TokenAuthentication.class.equals(authentication);
    }
}
```

### Add the second filter and provider to the configuration <a name="add_second_to_configuration"></a>

The last thing is to add these new filters and providers to the configuration:

```java
@Configuration
public class ProjectBeanConfiguration extends WebSecurityConfigurerAdapter {
    // ...

    // second filter and provider
    @Bean
    public TokenAuthFilter tokenAuthFilter() {
        // not using Autowired to avoid circular dependency issue
        return new TokenAuthFilter();
    }
    @Autowired
    private TokenAuthProvider tokenAuthProvider;

	// ...

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.authenticationProvider(usernamePassswordAuthProvider)
                .authenticationProvider(otpAuthProvider)
                .authenticationProvider(tokenAuthProvider);
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.addFilterAt(usernamePasswordAuthFilter, BasicAuthenticationFilter.class)
            .addFilterAfter(tokenAuthFilter, BasicAuthenticationFilter.class);
    }
}
```

I will continue with the next one ...
