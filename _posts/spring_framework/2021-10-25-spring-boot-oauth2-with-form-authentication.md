---
layout: post
title: Spring Boot OAuth2, Postgresql, Thymeleaf with Form based Authentication
date: 2021-10-25 22:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

Let's implement oauth2 practical implementation with Spring Boot using PostgreSQL and Thymeleaf. In this application: 

- We will use PostgreSQL to store user information.
- Thymeleaf for template engine
- Basic Authentication(Form based) for another way to logged-in our users.

At the end our clients will be able to login our web application both Basic Authentication (default security layer implemented by Spring Security) and OAuth2 (using Google Authorization)

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

There are three urls in our web application:

```java
public interface Urls {
    String INDEX = "/";
    String LOGGED_IN_PAGE = "/home";
    String REGISTER_PAGE = "/register";
}
```

- `/` => If user(s) is already logged-in, we will show user's profile page, otherwise we will show login page.
- `/register`=> We will show register form to store user via traditional way.

In the login page, there will be also sign in with Google button:

<img src="/assets/spring/oauth2/with_form_based_auth/login_page.png" alt="login_page" />



Here is the register page:

<img src="/assets/spring/oauth2/with_form_based_auth/register.png" alt="register_page" />



Here are the required dependencies for this project:

```xml
<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-thymeleaf</artifactId>
		</dependency>
    	<dependency>
			<groupId>org.thymeleaf.extras</groupId>
			<artifactId>thymeleaf-extras-springsecurity5</artifactId>
		</dependency>
		<dependency>
			<groupId>org.thymeleaf</groupId>
			<artifactId>thymeleaf-spring5</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-oauth2-client</artifactId>
		</dependency>

    	<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
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
</dependencies>
```

> Quick note: You should also need to change the following properties:
>
> ```properties
> spring.datasource.url=jdbc:postgresql://localhost:5432/test
> spring.datasource.username=postgres
> spring.datasource.password=1234
> ```

We have only one controller:

```java
@Controller
public class OAuth2Controller {
    private static final Logger logger = LoggerFactory.getLogger(OAuth2Controller.class);

    @GetMapping(value = Urls.INDEX)
    public String getIndexPage() {
        if (CheckUserAuthentication.isUserAuthenticated()) {
            return "logged_in_users";
        }
        return "not_logged_in_page";
    }

    @GetMapping(value = Urls.LOGGED_IN_PAGE)
    public String loggedInUserPage(Model model) {
        SecurityContext securityContext = SecurityContextHolder.getContext();
        Authentication currentUser = securityContext.getAuthentication();
        SecureUser loggedInUser = (SecureUser) currentUser.getPrincipal();
        logger.info("Logged-in user: {}", loggedInUser);
        return "logged_in_users";
    }

    @GetMapping(value = Urls.REGISTER_PAGE)
    public String openLoginPage() {
        if (CheckUserAuthentication.isUserAuthenticated()) {
            return "logged_in_users";
        }
        return "register_page";
    }
}
```

Also we are going to store user information in the table called `users`:

```java
@Table(name = "users")
@Entity
public class UserDTO {

    @Id
    @GeneratedValue(strategy =  GenerationType.IDENTITY)
    private long id;

    @Enumerated(EnumType.STRING)
    @Column(name = "provider")
    private Provider provider;

    @Column(name = "email")
    private String email;

    @Column(name = "role")
    private String role;
}
```

## Github Link

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/spring-security-examples/tree/master/spring-security-oauth2)


We will first implement OAuth2, then we are going to update with Basic Authentication. Then, let's start with oAuth2:

## Implementing OAuth2

If you recall my previous blogs about Spring Security, I already drew the big picture of the [spring security](https://mehmetozanguven.github.io/spring/2020/12/29/spring-security-1-basic-concepts.html). Basically we need to define three things:

- Create ClientRegistration for google
- Create UserService
- Create Password Encoder

But for OAuth2, we don't need to define passwordEncoder. 

### 1) Create ClientRegistration

Let's create a `SecurityConfiguration`:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    private static final String CLIENT_SECRET = "your_client_secreut";
    private static final String CLIENT_ID = "your_client_id";

    // ...

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

With this configuration, we are basically saying that: "Hey Spring boot if you get any request with this url: `/oauth2/authorization/google` , redirects that user to the Google Sign-In Page with the client secret and client id (provided by google api console)"

### 2) Create OAuth2 UserService

While we were implementing basic authentication, Spring Security forced us to create object with type `UserDetails`. Same thing will be applied for the OAuth2 login. 

Spring Security forces us to return object with type of `OAuth2User` if we are going to login with oauth2.

Spring Security forces us to return object with type of `OidcUser` (which extends `OAuth2User`) if we are going to login with oauth2 with OpenID Connect 1.0. 

> **What is OpenID Connect?**
>
> OpenID Connect 1.0 is a simple identity layer on top of the OAuth 2.0 protocol. 

At the end `OAuth2UserService` should return `OAuth2User`or `OidcUser`.  

OAuth2 UserService will apply the followings:

- Gets the information from google authorization, such as gmail address.
- Checks whether this email is in the database.
- If user isn't defined in the database: 
  - Creates a new record for that user 
  - Saves new user to the database.
  - Return appropriate object
- If user is already defined in the database
  - Return appropriate object

Because Google supports OpenID 1.0 identity layer. Our method should implement `OAuth2UserService` with also support openID as well. Spring Security provides a class called ` OidcUserService` for openId connect 1.0 provider's.

Here is the class for openId (we are going to override `loadUser` method):

```java
// An implementation of an {OAuth2UserService} that supports OpenID Connect 1.0 Provider's.
public class OidcUserService implements OAuth2UserService<OidcUserRequest, OidcUser> {
    
    public OidcUser loadUser(OidcUserRequest userRequest) throws OAuth2AuthenticationException {
    }
}
```
Here is the our oidcServer (every time user allows to login with Google, `loadUser()` method will be called):

```java
@Service
public class MyOidcService extends OidcUserService {
    private static final Logger logger = LoggerFactory.getLogger(MyOidcService.class);

    @Autowired
    private UserService userService;

    @Override
    public OidcUser loadUser(OidcUserRequest userRequest) throws OAuth2AuthenticationException {
        OidcUser oidcUser = super.loadUser(userRequest);
        try {
            return userService.findUser(userRequest, oidcUser);
        } catch (AuthenticationException ex) {
            throw ex;
        } catch (Exception ex) {
            logger.error("Error", ex);
            // Throwing an instance of AuthenticationException will trigger the
            // OAuth2AuthenticationFailureHandler
            throw new OAuth2CustomException(ex.getMessage());
        }
    }
}
```

In this line: ` OidcUser oidcUser = super.loadUser(userRequest);` , we are returning `OAuth2User` after obtaining the user attributes from the provider(s) (attributes can be gmail address etc..)

Then, `userService.findUser(userRequest, oidcUser)`, we will check this user is our db or not, then we will return object which implements `OidcUser`.

Before diving into `userService`, let's first register `MyOidcServer`in the security configuration.

### Register OAuth2 Service

```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    private static final String CLIENT_SECRET = "your_client_secreut";
    private static final String CLIENT_ID = "your_client_id";

    @Autowired
    private MyOidcService myOidcService;
	// ...

    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http.oauth2Login()
                .defaultSuccessUrl(Urls.LOGGED_IN_PAGE, true)
                .userInfoEndpoint().oidcUserService(myOidcService);
    }
}
```

`userInfoEndpoint()` returns configurer to register our oidcService.

### Checks Gmail in the database

```java
@Service
public class UserService {
    private static final Logger logger = LoggerFactory.getLogger(UserService.class);
    @Autowired
    private UserRepository userRepository;

   	// ...
    @Transactional
    public SecureUser findUser(OidcUserRequest userRequest, OidcUser oidcUser) {
        GoogleOAuth2Request googleOAuth2Request = new GoogleOAuth2Request(userRequest.getClientRegistration().getRegistrationId(), oidcUser.getAttributes());

        Optional<UserDTO> userInDb = findByUsername(googleOAuth2Request.getEmail());
        if (userInDb.isEmpty()) {
            logger.info("new user with email: {}", googleOAuth2Request.getEmail());
            UserDTO userDTO = createNewUser(googleOAuth2Request.getEmail(), Provider.GOOGLE);
            saveNewUser(userDTO);
            return SecureUser.createUser(oidcUser, userDTO);
        } else {
            return SecureUser.createUser(oidcUser, userInDb.get());
        }
    }
}
```

We have two providers:

``` java
public enum Provider {
    LOCAL("local"), GOOGLE("google");
	// ...
}
```

`GoogleOAuth2Request` is a kind of utility method to obtain email from oauth2 provider:

 ```java
 public class GoogleOAuth2Request {
 
     private String registrationId;
     private Map<String, Object> attributes; // attributes from Google AuthServer
 
     public GoogleOAuth2Request(String registrationId, Map<String, Object> attributes) {
         this.registrationId = registrationId;
         this.attributes = attributes;
     }
 
     public String getId() {
         return (String) attributes.get("sub");
     }
 
     public String getName() {
         return (String) attributes.get("name");
     }
 
     public String getEmail() {
         // returns gmail
         return (String) attributes.get("email");
     }
 
     public String getImageUrl() {
         return (String) attributes.get("picture");
     }
 }
 ```

`SecureUser`is an implementation of `OidcUser`:

```java
public interface OidcUser extends OAuth2User, IdTokenClaimAccessor {
	Map<String, Object> getClaims();
	OidcUserInfo getUserInfo();
	OidcIdToken getIdToken();
}

public class SecureUser implements OidcUser {
	// ...
    public static SecureUser createUser(OidcUser oidcUser, UserDTO userDTO) {
        SecureUser myOidcUser = new SecureUser();
        myOidcUser.setClaims(oidcUser.getClaims());
        myOidcUser.setUserInfo(oidcUser.getUserInfo());
        myOidcUser.setIdToken(oidcUser.getIdToken());
        myOidcUser.setAttributes(oidcUser.getAttributes());

        SimpleGrantedAuthority readAuthority = new SimpleGrantedAuthority(userDTO.getRole());
        myOidcUser.setAuthorities(Collections.singleton(readAuthority));
        myOidcUser.setName(userDTO.getEmail());

        return myOidcUser;
    }
	// ...
}
```

> We must also check the Provider, if User is already defined in the database. Because same username(same emails) may also try to login with Google and Local

Now if we click Login with Google button, we will be redirected to the Google Sign in Page, after all we will be redirected to the logged_in page in our spring application:

<img src="/assets/spring/oauth2/with_form_based_auth/google_logged_in.png" alt="google_logged_in" />

Our OAuth2 implementation is fully functional. Let's add form based authentication.

---

## Add Form Based Authentication

Here are the steps to implement form based authentication: (I am not going into detail why we need a new UserDetailsService etc.. You can find the reason from my [previous blogs](https://mehmetozanguven.github.io/spring-framework/))

- Create a new UserDetailsService to find out the user in the database
- Register new UserDetailsService using SecurityConfiguration
- Create PasswordEncoder to encode password
- Setup form based authentication using SecurityConfiguration

But before applying these steps, we first need to add **password** field in the table and the entity.

### Update Database and Entity with Password Column

First add new column called password:

```sql
ALTER TABLE users ADD COLUMN password varchar(100);
```

Second update the user entity:

```java
@Table(name = "users")
@Entity
public class UserDTO {
	// ...
    // new field
    @Column(name = "password")
    private String password;
}
```



### 1) Create UserDetailsService

We should create another service but in this time, this service must implement `UserDetailsService`:

```java
public class MyLocalUserDetailsService implements UserDetailsService {

    @Autowired
    private UserService userService;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Optional<UserDTO> userInDb = userService.findByUsername(username);
        if (userInDb.isEmpty()) {
            throw new UsernameNotFoundException("User with this username: " + username + " not found");
        }
        return SecureUser.createFromBasicAuthentication(userInDb.get());
    }
}
```

Because we should return `UserDetails`, then our `SecureUser`object must also implement `UserDetails` interface

```java
public class SecureUser implements OidcUser, UserDetails {
    private Map<String, Object> claims;
    private OidcUserInfo userInfo;
    private OidcIdToken idToken;
    private Map<String, Object> attributes;
    private Collection<SimpleGrantedAuthority> authorities;
    private String name;

    // Common for OAuth2 and BasicAuthentication
    private String provider;

    // Fields for UserDetails
    private String password;
    private String username;

    public static SecureUser createFromBasicAuthentication(UserDTO userDTO) {
        SecureUser basicAuth = new SecureUser();
        basicAuth.setProvider(Provider.LOCAL.name);
        basicAuth.setUsername(userDTO.getEmail());
        basicAuth.setPassword(userDTO.getPassword());

        return basicAuth;
    }
    // ...
}
```

### 2) Register UserDetailsService

We should create a bean from our custom user details service. Then Spring Boot can be able to use our implementation instead of the default one:

``` java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

    // ...
    @Bean
    public UserDetailsService userDetailsService() {
        return new MyLocalUserDetailsService();
    }
}
```

### 3) Create passwordEncoder

Because we are implementing custom user details service. We must also create password encoder as well.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    // ...
    @Bean
    public UserDetailsService userDetailsService() {
        return new MyLocalUserDetailsService();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 4) Setup form based authentication

```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    // ...
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // permit specific urls, (js, css, etc...)
        http.authorizeRequests().antMatchers(permittedUrls()).permitAll();

        // anything than the permitted urls must be protected
        http.authorizeRequests().anyRequest().authenticated();

        // configure logout functionality,
        // after logout has occurred, redirects user to the "/"
        http.logout(logout -> logout
                .logoutRequestMatcher(new AntPathRequestMatcher("/logout"))
                .logoutSuccessUrl(Urls.INDEX)
                .deleteCookies("JSESSIONID")
                .invalidateHttpSession(true)
                .clearAuthentication(true)
        );

        // oauth2 configuration,
        // after successful login, redirects user to the "/home"
        http.oauth2Login()
                .defaultSuccessUrl(Urls.LOGGED_IN_PAGE, true)
                .userInfoEndpoint().oidcUserService(myOidcService);

        // Setup form login authentication
        http.formLogin().loginPage(Urls.INDEX).successForwardUrl(Urls.LOGGED_IN_PAGE);
    }
}
```

## Testing the application

I have created new user from register form. If I run select query for  `users` table:

```sql
select * from users;
-[ RECORD 1 ]----------------------------------------------------------
id       | 1
email    | mehmetozanguven@gmail.com
provider | GOOGLE
role     | READ
password | 
-[ RECORD 2 ]----------------------------------------------------------
id       | 2
email    | ozan@ozan.com
provider | LOCAL
role     | READ
password | $2a$10$A9rodo18nsrI.gX5D7O1L.JOi4/s48.THHC3K3vm1YlG59hDV19NW
```

As you can see we have two records one with google and another with local(with the encoded password way)

Finally here is the `/home` page via local provider:

<img src="/assets/spring/oauth2/with_form_based_auth/form_Based_login.png" alt="form_Based_login" />

## Conclusion

In this blog, we implemented both OAuth2 and form based login in the same application. We implemented both authentication processes step by step. Both authentication processes force us to return specific type of user. In OAuth2, Spring security forces us to return user object with type of OidcUser. In form based authentication, Spring Security forces us to return user object with UserDetails.

 For OAuth2:

1. Create client registraion and put into the client repository.
2. Create oauth2 service for google sign in, service must also provide support for open ID. (Spring Security provides `OidcUserService` base class for that)
3. Register custom oauth2 service via `.userInfoEndpoint().oidcUserService(myOidcService);` 
4. Checks the user is in the database (via gmail address or something else). If it is in the db, return from db, otherwise, create a new record for that user.

For Login form based authentication:

1. Create UserDetailsService (class which extends `UserDetailsService`)
2. Register UserDetailsService (basically create custom user details service as a bean)
3. Because we are creating custom user details service, we must also create password encoder
4. OidcUserServiceSetup form based authentication in the security configuration.



Wait for the next blog ...
