---
layout: post
title: Spring Boot JWT Setup with Frontend (VueJs)
date: 2021-07-12 14:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

In this post, we are going to setup spring boot rest project with using JWT. we will also integrate the our spring boot application with the frontend (in this case I am going to use VueJS).

Topics are:

- [**Github Link**](#github_link)
- [**Project Setup**](#project_Setup)
  - [**Dependencies**](#dependencies)
  - [**PostgreSQL Setup**](#postgresql_setup)
- [**What is the JSON Web Token (JWT)**](#what_is_jwt)
- [**Security Configuration**](#security_configuration)
  - [**CORS Policy setup**](#cors_setup)
  - [**UserDetailsService**](#user_details_service)
  - [**BCryptPasswordEncoder Bean**](#password_encoder)
  - [**Allow Register and Login Endpoints**](#allow_register_and_login_endpoints)
- [**Implementing Register Endpoint**](#implement_register_endpoint)
  - [**Validate Register Request**](#validate_register_request)
  - [**Delegate Request to the RegisterService & Wrap the Service Response**](#delegate_to_register_service)
- [**Implementing Register Service**](#implement_register_service)
  - [**Check the Username**](#check_username)
  - [**Encrypt user's password & save into db**](#save_to_db)
    - [**Error Invalid CSRF token found**](#csrf_invalid_token)
- [**Implementing Login Endpoint**](#implement_login_endpoint)
  - [**Validate Login Request**](#validate_login_request)
  - [**Delegate Request to the LoginService & Wrap the Service Response**](#delegate_to_login_service)
- [**Implementing Login Service**](#implement_login_service)
  - [**Delegate the request to the `AuthenticationManager`**](#delegate_to_auth_manager)
  - [**Set the fully authenticated user to the security context**](#set_auth_user)
  - [**Generate JWT with Username**](#generate_jwt)
  - [**Return the login response**](#return_login_response)
- [**Implementing AuthTokenFilter**](#implement_auth_filter)
  - [**Intercept the all incoming requests**](#intercept_all_requests)
  - [**Get JWT from the request**](#get_jwt_from_request)
  - [**Validate JWT**](#validate_jwt)
  - [**Get the username from JWT and find the authenticated user**](#get_username_from_jwt)
  - [**Forward the request to the next filter**](#forward_request_to_next_filter)
- [**Adding AuthTokenFilter to our application**](#add_auth_filter_to_app)
  - [**Reason for `addFilterBefore`**](#filter_before)
- [**Create Dummy Protected Controller**](#dummy_controller)
- [**Front-end Setup**](#frontend-setup)

## Github Link <a name="github_link"></a>

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/spring-boot-examples/tree/master/spring-boot-jwt-example)

## Project Setup <a name="project_Setup"></a>

### Dependencies <a name="dependencies">

Please create a new spring boot project with the following dependencies:

- Spring Web
- Spring Security
- Spring Data JPA
- Spring Validation
- PostgreSQL Driver
- io.jsonewbtoken (JSON web token support for the JVM) (Maven link: https://mvnrepository.com/artifact/io.jsonwebtoken/jjwt)

### PostgreSQL Setup <a name="postgresql_setup">

You must change the postgresql configuration inside the `application.properties` file as your needs.

> Change the url, username and password

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/testdatabase
spring.datasource.username=postgres
spring.datasource.password=1234
# For localdb, it would be good to re-create all-tables,
# because ddl-auto = update, for the first run, all tables will be created automatically
spring.jpa.hibernate.ddl-auto=update
logging.level.org.springframework.security=DEBUG
```

## What is the JSON Web Token (JWT) <a name="what_is_jwt"></a>

JSON Web Token (JWT) is an open standard ([RFC 7519](https://tools.ietf.org/html/rfc7519)) that defines a compact and self-contained way for securely transmitting information between parties as a JSON object. This information can be verified and trusted because it is digitally signed.

You should use JWT for:

- **Authorization**: Most common scenario. After user sends correct username&password and logged-in to they system, other subsequent request include the JWT, allowing the user to access our methods(endpoints etc..)
- **Information Exchange**: Because it is digitally signed. If someone signed JWT with its private key, then it can only be opened by its public key. Therefore you will guarantee that message from user is actually belong to him/her.

JWT has its own structure but i am not going into detail of it. You may look at the https://jwt.io/introduction

## Security Configuration <a name="security_configuration"></a>

In the project, because we will use our database, then we need to implement our userDetailsService via implementing `UserDetailsService` interface.

As I said my previous post, If we create our UserDetailsService, we must also create a PasswordEncoder.

> I am going to use `BCryptPasswordEncoder`

### CORS Policy Setup <a name ="cors_setup"></a>

Because we have two different origins (one for running on localhost:8080-spring boot-, and one for running on localhost:8081-vuejs-), we should enable cross origin request between these origins. Otherwise client ajax request (I will use axios) will fail. For the sake of simplicity, I allowed all the origins:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
	// ...
    @Bean
    public CorsFilter corsFilter() {
        CorsConfiguration corsConfiguration = new CorsConfiguration();
        corsConfiguration.setAllowedOrigins(Collections.singletonList(CorsConfiguration.ALL));
        corsConfiguration.setAllowedMethods(Collections.singletonList(CorsConfiguration.ALL));
        corsConfiguration.setAllowedHeaders(Collections.singletonList(CorsConfiguration.ALL));
        corsConfiguration.setMaxAge(Duration.ofMinutes(10));
        corsConfiguration.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", corsConfiguration);
        return new CorsFilter(source);
    }
	// ...
}
```

### Make CSRF token accessible via Cookies <a name ="csrf_cookie_setup"></a>

I will allow spring to set csrf token in the cookie, otherwise I can't send the csrf token for the following requests.

> Actually in the front-end, I will store the JWT in the local-storage, therefore I could disable csrf protection at all. You can just reference my project to see "how to enable CSRF with rest communication"

```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
	// ...
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf(c -> {
            c.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse());
        });
        // ...
    }
}
```

### UserDetailsService <a name="user_details_service"></a>

UserDetailsService will only get the username from the authentication provider and send it to the `userRepository` to get one User from database.

```java
@Service
public class MyUserDetailService implements UserDetailsService {
    private final UserRepository userRepository;

    public MyUserDetailService(@Autowired UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Optional<UserDTO> userInDb = userRepository.findByUsername(username);
        UserDTO userDTO = userInDb.orElseThrow(() -> new UsernameNotFoundException("Not Found in DB"));
        return new SecureUser(userDTO);
    }
}
```

If repository returns the User from the database, we should wrap it with new object. And this new object must extend the `UserDetails`. Otherwise spring security can not know authenticated User. That is the reason why I created a new class called `SecureUser`:

```java
public class SecureUser implements UserDetails {
    private List<SimpleGrantedAuthority> userRoles;
    private String password;
    private String username;


    public SecureUser(UserDTO userDTO) {
       this.userRoles = new ArrayList<>();
       userDTO.getUserRoles().forEach(userRoleDTO -> this.userRoles.add(new SimpleGrantedAuthority(userRoleDTO.getRole())));

       this.password = userDTO.getPassword();
       this.username = userDTO.getUsername();
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return this.userRoles;
    }

    @Override
    public String getPassword() {
        return this.password;
    }

    @Override
    public String getUsername() {
        return this.username;
    }

    // ...
}
```

Because we created `MyUserDetailService` as a `Service` bean, Spring will automatically use our `UserDetailsService`

### BCryptPasswordEncoder Bean <a name="password_encoder"></a>

We also must override the default configuration when we create custom UserDetailsService. This step is just to creating a bean in the configuration class. Here is the security configuration:

```java
@Configuration
public class SecurityConfiguration {
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

If you run the project in this setup, your all requests will return "Unauthorized" error message with 401 status code. Because Spring Security protects all the endpoints. In other words it means, we aren't authenticated.

```bash
$ curl -X GET http://localhost:8080
```

Response:

```json
{
  "timestamp": "2021-07-10T15:59:56.462+00:00",
  "status": 401,
  "error": "Unauthorized",
  "message": "",
  "path": "/"
}
```

We must adjust some endpoints to pass spring security, These endpoints could be login and register endpoints

### Allow Register and Login Endpoints <a name="allow_register_and_login_endpoints"></a>

We should switch off the default web application security configuration and enable the our security configuration. We can achieve adding `@EnableWebSecuirty` annotation and extending with `WebSecurityConfigurerAdapter`

After that we need to specify which endpoint(s) should not be protected by Spring Security. Here is the SecurityConfiguration:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    private String[] allowedEndpoints() {
        return new String[] {
                "/api/login",
                "/api/register"
        };
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.cors().and()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()
                .authorizeRequests()
                .antMatchers(allowedEndpoints()).permitAll()
                .anyRequest().authenticated();
    }
}
```

Let's send it the same request:

```bash
$ curl -X GET http://localhost:8080
```

```json
{
  "timestamp": "2021-07-10T17:37:19.070+00:00",
  "status": 403,
  "error": "Forbidden",
  "message": "",
  "path": "/"
}
```

Basically it means that we are authenticated (in this case Anonymous) but Anonymous user has no permission to send request to the endpoint `http://localhost:8080`

To test the login and register endpoint, you may create a dummy methods:

> In the sake of simplicity, I just annotated these endpoints as GET requests.

```java
@RestController
@RequestMapping("/api")
public class LoginController {

    @GetMapping("/login")
    public ResponseEntity<?> doRegister() {
        return ResponseEntity.ok("login endpoint");
    }
}
```

```java
@RestController
@RequestMapping("/api")
public class RegisterController {

    @GetMapping("/register")
    public ResponseEntity<?> doRegister() {
        return ResponseEntity.ok("register endpoint");
    }
}
```

Now send request to the login or register endpoint:

```bash
$ curl -X GET http://localhost:8080/api/register
```

Response:

```wiki
register endpoint
```

We got a response, because we authenticated as AnonymousUser, also we have a permission to send request `/api/login` & `/api/register`

We should first sign-up new user. Therefore we need to implement our register endpoint.

## Implementing Register Endpoint <a name="implement_register_endpoint"></a>

Let's summarize the tasks when someone hits the register endpoints. The register endpoint have to perform the following actions:

- It will validate the request (this can be done by @Valid annotation)
- If request is valid, then it will delegate the request to the `RegisterService`
- Wrap the service response with `ResponseEntity`

### Validate Register Request <a name="validate_register_request"></a>

This step will be easy with annotations

```java
public class RegisterRequest {
    @NotBlank(message = "Username can not be empty")
    private String username;

    @NotBlank(message = "Password can not be empty")
    private String password;
    // getters and setters...
}
```

```java
@RestController
@RequestMapping("/api")
public class RegisterController {

    @PostMapping("/register")
    public ResponseEntity<?> doRegister(@RequestBody @Valid RegisterRequest registerRequest) {
        return ResponseEntity.ok("register endpoint");
    }
}
```

### Delegate Request to the RegisterService & Wrap the Service Response <a name="delegate_to_register_service"></a>

Simple:

```java
@RestController
@RequestMapping("/api")
public class RegisterController {
    private final RegisterService registerService;

    public RegisterController(@Autowired RegisterService registerService) {
        this.registerService = registerService;
    }

    @PostMapping("/register")
    public ResponseEntity<?> doRegister(@RequestBody @Valid RegisterRequest registerRequest) {
        RegisterResponse registerResponse = registerService.doRegister(registerRequest);
        return ResponseEntity.ok(registerResponse);
    }
}
```

## Implementing Register Service <a name="implement_register_service"></a>

The register service have to perform the following actions:

- First it checks the username, if the username is in our database, it throws an exception to indicate this user has already registered.

- It will encrypt the user password with the current password encoder (in our example it will be BCryptPasswordEncoder)

### Check the Username <a name="check_username"></a>

I believe code is self-explanatory.

> If you want to return specific response when user is not found, you should wait (or search on the internet) for my post related to that.

```java
@Service
public class RegisterServiceImpl implements RegisterService {
    private final PasswordEncoder passwordEncoder;
    private final RegisterRepository registerRepository;

    public RegisterServiceImpl(@Autowired PasswordEncoder passwordEncoder, @Autowired RegisterRepository registerRepository) {
        this.passwordEncoder = passwordEncoder;
        this.registerRepository = registerRepository;
    }

    @Transactional
    @Override
    public RegisterResponse doRegister(RegisterRequest registerRequest) {
        Optional<UserDTO> isUserInDb = registerRepository.findByUsername(registerRequest.getUsername());
        if (isUserInDb.isPresent()) {
            throw new RuntimeException("User has already in db");
        }
		// ...
    }
}
```

### Encrypt user's password & save into db <a name="save_to_db"></a>

```java
@Service
public class RegisterServiceImpl implements RegisterService {
    // ...
    @Transactional
    @Override
    public RegisterResponse doRegister(RegisterRequest registerRequest) {
       	// ...
        UserDTO newUser = mapNewRegisterToNewUser(registerRequest);
        registerRepository.save(newUser);
        RegisterResponse registerResponse = new RegisterResponse();
        registerResponse.setRegistered(true);
        registerResponse.setMesssage("User was saved");
        return registerResponse;
    }

    private UserDTO mapNewRegisterToNewUser(RegisterRequest registerRequest) {
        UserDTO userDTO = new UserDTO();
        userDTO.setUsername(registerRequest.getUsername());
        userDTO.setPassword(passwordEncoder.encode(registerRequest.getPassword()));

        UserRoleDTO basicRole = new UserRoleDTO();
        basicRole.setRole("ROL_BASIC");
        basicRole.setUserDTO(userDTO);

        userDTO.getUserRoles().add(basicRole);
        return userDTO;
    }
}
```

Let's try to send a request:

```bash
$ curl -X POST  -H "Content-Type: application/json" -d '{ "username": "test",  "password": "1234"}'  http://localhost:8080/api/register
```

You will not see any response. Check the logs in the console:

```wiki
 o.s.security.web.csrf.CsrfFilter         : Invalid CSRF token found for http://localhost:8080/api/register
 // ...
org.springframework.security.access.AccessDeniedException: Access is denied
	at org.springframework.security.access.vote.AffirmativeBased.decide(AffirmativeBased.java:84) ~[spring-security-core-5.3.5.RELEASE.jar:5.3.5.RELEASE]
	at org.springframework.security.access.intercept.AbstractSecurityInterceptor.beforeInvocation(AbstractSecurityInterceptor.java:233) ~[spring-security-core-5.3.5.RELEASE.jar:5.3.5.RELEASE]
	// ..
```

Spring Security will protect all endpoints(state-changing operations) for CSRF attacks by default. But we can allow the Spring Security to pass the register endpoint.

#### Error Invalid CSRF token found <a name="csrf_invalid_token"></a>

If you remember my previous post about [CSRF protection](https://mehmetozanguven.github.io/spring/2021/05/16/spring-security-8-csrf-setup.html) , you have a chance to disable CSRF protection for specific endpoint(s)

Just update security configuration:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

    // ...

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf(c -> {
            c.ignoringAntMatchers("/api/register");
        });

       // ...
    }
}
```

Re-run the project and send the request again, response will be:

```json
{ "messsage": "User was saved", "registered": true }
```

Take a look at the database:

```sql
testdatabase=# select * from users;
-[ RECORD 1 ]----------------------------------------------------------
id       | 1
password | $2a$10$IA3OlClwnzWxcxb2wXujhuZS/bMdkXga7yhzFswiN2UaUOlJpksAC
username | test
```

As you can see, we saved the user in the database.

If you send the same request, you will get a response like this:

```json
{
  "timestamp": "2021-07-10T19:35:13.141+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "message": "",
  "path": "/api/register"
}
```

Look at the console:

```wiki
java.lang.RuntimeException: User has already in db
	at com.mehmetozanguven.springbootjwtexample.service.register.RegisterServiceImpl.doRegister(RegisterServiceImpl.java:30) ~[classes/:na]
```

## Implementing Login Endpoint <a name="implement_login_endpoint"></a>

### Disable CSRF protection

First, I will disable the CSRF protection for the login endpoint.

> Because disabling csrf protection in the login endpoint may cause security vulnerabilities I am not sure about doing that.
>
> Disabling csrf in the login endpoint could be more dangerous rather than disabling on the register endpoint! Do your research then disable (or not disable) CSRF protection.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
	// ...
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf(c -> {
            c.ignoringAntMatchers(allowedEndpoints());
        });
        // ...
    }
}
```

The login endpoint have to perform the following actions:

- It will validate the request (this can be done by @Valid annotation)
- If request is valid, then it will delegate the request to the `LoginService`
- Wrap the service response with `ResponseEntity`

### Validate Login Request <a name="validate_login_request"></a>

Again!! This step will be easy with annotations

```java
public class LoginRequest {
    @NotBlank(message = "Username can not be empty")
    private String username;
    @NotBlank(message = "Password can not be empty")
    private String password;
    // getters and setters ..
}
```

```java
@RestController
@RequestMapping("/api")
public class LoginController {
    @PostMapping("/login")
    public ResponseEntity<?> doLogin(@RequestBody @Valid LoginRequest loginRequest) {
        // ...
    }
}
```

### Delegate Request to the LoginService & Wrap the Service Response <a name="delegate_to_login_service"></a>

```java
@RestController
@RequestMapping("/api")
public class LoginController {

    private final LoginService loginService;

    public LoginController(@Autowired LoginService loginService) {
        this.loginService = loginService;
    }


    @PostMapping("/login")
    public ResponseEntity<?> doLogin(@RequestBody @Valid LoginRequest loginRequest) {
        LoginResponse loginResponse = loginService.doLogin(loginRequest);
        return ResponseEntity.ok(loginResponse);
    }
}
```

## Implementing Login Service <a name="implement_login_service"></a>

The login service have to perform the following actions:

- Delegate the request to the authenticationManager to authenticate user.

> AuthenticationManager should return fully authenticated instance or throw an error.

- After getting fully authenticated instance, Login Service will set the fully authenticated user to the security context
- Generate jwt with username
- Return the login response

### Delegate the request to the `AuthenticationManager` <a name="delegate_to_auth_manager"></a>

We should first create an AuthenticationManager bean. For that we can use default authenticationManager object as bean:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
	// ...
    @Bean
    @Override
    protected AuthenticationManager authenticationManager() throws Exception {
        return super.authenticationManager();
    }
	// ...
}
```

```java
import org.springframework.security.authentication.AuthenticationManager;
// ...
@Service
public class LoginServiceImpl implements LoginService {
    private final AuthenticationManager authenticationManager;

    public LoginServiceImpl(@Autowired AuthenticationManager authenticationManager) {
        this.authenticationManager = authenticationManager;
    }

    @Override
    public LoginResponse doLogin(LoginRequest loginRequest) {
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(loginRequest.getUsername(), loginRequest.getPassword()));
    }
}
```

If you recall from previous blogs, `AuthenticationManager` will find the correct provider with the given `Authentication `. In this case we provided `UsernamePasswordAuthenticationToken` .

Here is the step by step what is happening:

- `AuthenticationManager` will try to find `AuthenticationProvider` for the `Authentication` instance.
- For `UsernamePasswordAuthenticationToken`, provider will be `DaoAuthenticationProvider`
- `AuthenticationManager` will call `Authentication authentication = provider.authenticate()` method.
- Provider will return fully authenticated object or throw an exception
- Because we have defined our `UserDetailsService` and `PasswordEncoder`. `DaoAuthenticationProvider` will use these beans.
- Provider will call the method: `UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);`

> `this.getUserDetailsService()` equals to our `UserDetailsService(MyUserDetailService)` . After all `MyUserDetailService#loadUserByUsername` will be called by the provider. And we will return `UserDetails` or throw an error

- After getting `UserDetails`, provider will check the request's password and password from the database. Provider will use our password encoder for that (`BCryptPasswordEncoder`)

```java
protected void additionalAuthenticationChecks(UserDetails userDetails,
			UsernamePasswordAuthenticationToken authentication)
			throws AuthenticationException {
    	// userDetails = MyUserDetailService#loadUserByUsername

    	// authentication.getCredentials() = loginRequest.getPassword()
		String presentedPassword = authentication.getCredentials().toString();
		if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
			logger.debug("Authentication failed: password does not match stored value");

			throw new BadCredentialsException(messages.getMessage(
					"AbstractUserDetailsAuthenticationProvider.badCredentials",
					"Bad credentials"));
		}
	}
```

### Set the fully authenticated user to the security context <a name="set_auth_user"></a>

Spring will store the authenticated object in the `SecurityContext`. You need this setup, if you want to get authenticated user anywhere in the code. Thus we should manually set the authenticated object.

```java
@Service
public class LoginServiceImpl implements LoginService {
	// ...
    @Override
    public LoginResponse doLogin(LoginRequest loginRequest) {
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(loginRequest.getUsername(), loginRequest.getPassword()));
        SecurityContextHolder.getContext().setAuthentication(authentication);
       	// ...
    }
}
```

### Generate JWT with Username <a name="generate_jwt"></a>

We only need to find authenticated username and generating JWT for the future requests. (User will send the jwt in the header)

> I also created JwtUtil class to validate and generate. But I won't explain how it works. You can search on the Internet.

```java
@Service
public class LoginServiceImpl implements LoginService {
  	// ...
    @Override
    public LoginResponse doLogin(LoginRequest loginRequest) {
        // ...
        SecureUser userDetails = (SecureUser) authentication.getPrincipal();
        String jwtToken = JwtUtil.generateJwtToken(userDetails.getUsername());
        // ...
    }
}

```

### Return the login response <a name="return_login_response"></a>

Response from the login service will only include JWT.

```java
@Service
public class LoginServiceImpl implements LoginService {
    // ...
    @Override
    public LoginResponse doLogin(LoginRequest loginRequest) {
        // ...
        LoginResponse loginResponse = new LoginResponse();
        loginResponse.setJwtToken(jwtToken);
        return loginResponse;
    }
}
```

Our login endpoint is ready to test. Send this request:

```bash
$ curl -X POST  -H "Content-Type: application/json" -d '{ "username": "test",  "password": "1234"}'  http://localhost:8080/api/login | jq .
```

Response:

```json
{
  "jwtToken": "eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJ0ZXN0IiwiaWF0IjoxNjI1OTUxMDczLCJleHAiOjE2MjU5NTQ2NzN9.wJq_Vu0ekWq02tKERNV5aEIiueKUA76UodLQVumqOPZXH_ZDlMGluE_yXJFltjjPF7-H83yVadozoiOfH0zfBg"
}
```

For invalid credentials:

```bash
$ curl -X POST  -H "Content-Type: application/json" -d '{ "username": "test",  "password": "12345"}'  http://localhost:8080/api/login | jq .
```

```json
{
  "timestamp": "2021-07-10T21:05:25.958+00:00",
  "status": 403,
  "error": "Forbidden",
  "message": "",
  "path": "/api/login"
}
```

Look at the console:

```wiki
// ...
org.springframework.security.authentication.BadCredentialsException: Bad credentials
	at org.springframework.security.authentication.dao.DaoAuthenticationProvider.additionalAuthenticationChecks
	// ...
```

After successful login, we should check the JWT for each subsequent request to verify the user. We somehow intercepts the all coming requests. This can be done by adding filter in our application. Let's implement this feature.

## Implementing AuthTokenFilter <a name="implement_auth_filter"></a>

This filter will have the following responsibilities:

- Intercept the all incoming requests
- Get JWT from the request
- Validate JWT.
- Get the username from JWT and find the authenticated user.
- Forward the request to the next filter

### Intercept the all incoming requests <a name="intercept_all_requests"></a>

Spring already provides a base class to intercept all requests which is `OncePerRequestFilter`. We need only to extend this class.

```java
public class AuthTokenFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {

    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) throws ServletException {
        return  request.getServletPath().equals("/api/login") ||
                request.getServletPath().equals("/api/register");
    }
}
```

### Get JWT from the request <a name="get_jwt_from_request"></a>

JWT will be send in the Header with the following format:

```wiki
Authorization: Bearer {JWT}
```

```java
public class AuthTokenFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        try {
            String jwt = parseJwt(request)
        } catch (Exception e) {
            logger.error("Exception: ", e);
        }
        filterChain.doFilter(request, response);
    }
	// ...

    private String parseJwt(HttpServletRequest request) {
        String headerAuth = request.getHeader("Authorization");

        if (StringUtils.hasText(headerAuth) && headerAuth.startsWith("Bearer ")) {
            return headerAuth.substring(7, headerAuth.length());
        }

        return null;
    }
}
```

### Validate JWT <a name="validate_jwt"></a>

I will use the method from `JwtUtil` to verify.

```java
public class AuthTokenFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        try {
            String jwt = parseJwt(request);
			if (jwt != null && JwtUtil.validateJwtToken(jwt)) {
				// JWT valid
            }
        } catch (Exception e) {
            logger.error("Exception: ", e);
        }
        filterChain.doFilter(request, response);
    }
}
```

### Get the username from JWT and find the authenticated user <a name="get_username_from_jwt"></a>

In this time, we don't need to call authenticationManager. Using the UserDetailsService will be enough:

```java
public class AuthTokenFilter extends OncePerRequestFilter {
    @Autowired
    private  MyUserDetailService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        try {
            // ...
            if (jwt != null && JwtUtil.validateJwtToken(jwt)) {
                String username = JwtUtil.getUsernameFromJwtToken(jwt);
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        }
        // ...
    }
}
```

After getting users from database with the `UserDetails` type:

- We are creating fully authenticated token:

```java
 UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
```

- Set the fully authenticated user to the security context:

```java
SecurityContextHolder.getContext().setAuthentication(authentication);
```

### Forward the request to the next filter <a name="forward_request_to_next_filter"></a>

Make sure that called the `filterChain.doFilter(request, response);` at the end.

```java
public class AuthTokenFilter extends OncePerRequestFilter {
	// ...
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        try {
          // ...
        } catch (Exception e) {
            // ...
        }
        filterChain.doFilter(request, response);
    }
}
```

## Adding AuthTokenFilter to our application <a name="add_auth_filter_to_app"></a>

We can add filter before `UsernamePasswordAuthenticationFilter.class`

```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
	// ...
    @Bean
    public AuthTokenFilter authTokenFilter() {
        return new AuthTokenFilter();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
       // ...
        http.addFilterBefore(authTokenFilter(), UsernamePasswordAuthenticationFilter.class);
    }
}
```

### Reason for `addFilterBefore` <a name="filter_before"></a>

In the Spring environment we have three options to put filter:

- `http.addFilter()` :

Adds a `Filter` that must be an instance of or extend one of the Filters provided within the Security framework. The method ensures that the ordering of the Filters is automatically taken care of. In other words, you can not use this method to add your custom filter. In our project, `AuthTokenFilter` is a custom filter and not extending any of the filters provided by Spring Security. Some of the known(or provided) filters: `ChannelProcessingFilter, SecurityContextPersistenceFilter, LogoutFilter, SessionManagementFilter...` All list can be found in this link: https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/config/annotation/web/HttpSecurityBuilder.html#addFilter(javax.servlet.Filter)

- `http.addFilterBefore` :

Allows adding a `Filter` before one of the known `Filter` classes. The known `Filter` instances are either a `Filter` listed in `addFilter(filter)` (click the link above) or a `Filter` that has already been added using `addFilterAfter` or `addFilterBefore`

In other words, we are free to insert our custom filter.

- `http.addFilterAfter` :

Allows adding a `Filter` before one of the known `Filter` classes. The known `Filter` instances are either a `Filter` listed in `addFilter(filter)` (click the link above) or a `Filter` that has already been added using `addFilterAfter` or `addFilterBefore`

In other words, we are free to insert our custom filter.

The only part is to test this configuration with the protected resource(s)/controller(s). Let's create dummy endpoint and test it.

## Create Dummy Protected Controller <a name="dummy_controller"></a>

This controller will return simple value to demonstrate only valid jwt can access it.

```java
@RestController
@RequestMapping("/api")
public class ProtectedController {

    @GetMapping("/protected")
    public ResponseEntity<?> getProtectedResource() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        String currentPrincipalName = authentication.getName();
        return ResponseEntity.ok("Hello Logged-In User:" + currentPrincipalName + ", you can access this resource, because JWT was valid");
    }
}
```

First login into the system and get the JWT:

```bash
$ curl -X POST  -H "Content-Type: application/json" -d '{ "username": "test",  "password": "1234"}'  http://localhost:8080/api/login
```

Response:

```json
{
  "jwtToken": "eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJ0ZXN0IiwiaWF0IjoxNjI2MTAyMzk5LCJleHAiOjE2MjYxMDU5OTl9.wfvvukr9h5e7KS7YDqr3xFxv9iydlPXT8wW50zLaHWkTRF-gW_je_G7QwOWEshV1YCRrC9RQUViAsebksMI3CQ"
}
```

Now we can use this JWT in the header for the subsequent request(s):

```bash
$ curl -X GET  -H "Content-Type: application/json" -H "Authorization: Bearer {{JWT}} " http://localhost:8080/api/protected

Hello Logged-In User:test, you can access this resource, because JWT was valid
```

> Quick note: We don't setup anything related to the CSRF protection. Because CSRF protection is only needed for state changing operation(s). And also in the spring security `CsrfFilter` will not be applied when http method is one of the following ones:
>
> `"GET", "HEAD", "TRACE", "OPTIONS"`

With invalid JWT, you will get the response:

```json
{
  "timestamp": "2021-07-12T15:11:00.026+00:00",
  "status": 403,
  "error": "Forbidden",
  "message": "",
  "path": "/api/protected"
}
```

For invalid JWT, look at the console:

```wiki
 JwtUtil : Invalid JWT signature: JWT signature does not match locally computed signature. JWT validity cannot be asserted and should not be trusted.
```

For a final step, we can customize the default error response implementing `AuthenticationEntryPoint`. We can do that in another post.

## Front-end Setup <a name="frontend-setup"></a>

Unfortunately, there is really hard to explain front-end implementation step by step. Because I don't have enough knowledge about front-end side. I will just give you a [github link](https://github.com/mehmetozanguven/spring-boot-jwt-frontend)
