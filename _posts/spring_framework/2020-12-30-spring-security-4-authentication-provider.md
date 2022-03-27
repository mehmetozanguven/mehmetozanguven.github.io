---
layout: post
title: Spring Security -- 4) Implementing Custom Authentication Provider
date: 2020-12-30 19:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
permalink: spring-security-4-authentication-provider
---

In this post, I am going to answer to this question "what is the **Authentication Provider**" and I am going to implement a project includes custom authentication provider.

Topics are:

- [**Github Link**](#github_link)
- [**Overall Architecture in one picture**](#overall_architecture)
- [**Why you would need a custom authentication provider**](#why_custom_provider)
- [**Default Project Setup**](#default_project_setup)
- [**Creating Custom Authentication Provider**](#custom_authentication_provider)
  - `authenticate()` and `supports()` method
  - `MyCustomAuthenticationProvider` class
  - Adding the CustomProvider to the configuration

## Github Link <a name="github_link"></a>

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/spring-security-examples/tree/master/spring-security-authentication-provider)

## Overall Architecture in one picture <a name="overall_architecture"></a>

<img src="/assets/spring/security/authentication_provider/customauthenticationprovider.png" alt="customauthenticationprovider.png" />

## Why you would need a custom authentication provider? <a name="why_custom_provider"></a>

You might need an architecture which authentication process can not be done via username and password. In other words you might need to implement your own authentication logic. In that case, you can implement custom authentication provider.

## Default Project Setup <a name="default_project_setup"></a>

In default I have a dummy user creating via InMemoryUserDetailsManager

You only need these dependencies:

```xml
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
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

Controller:

```java
package com.mehmetozanguven.springsecurityauthenticationprovider.controller;

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

Configuration:

```java
package com.mehmetozanguven.springsecurityauthenticationprovider.config;

@Configuration
public class ProjectBeanConfiguration {

    @Bean
    public UserDetailsService userDetailsService(){
        InMemoryUserDetailsManager inMemoryUserDetailsManager = new InMemoryUserDetailsManager();
        UserDetails dummyUser = User.withUsername("dummy_user")
                                    .password("1234")
                                    .authorities("read")
                                    .build();
        inMemoryUserDetailsManager.createUser(dummyUser);
        return inMemoryUserDetailsManager;
    }

    @Bean
    public PasswordEncoder passwordEncoder(){
        return NoOpPasswordEncoder.getInstance();
    }
}
```

## Creating Custom Authentication Provider <a name="custom_authentication_provider"></a>

Instead of default `AuthenticationProvider` provided by Spring, let's use a custom one.

In general `AuthenticationProvider` contains two methods: `authenticate()` contains the authentication logic and `supports()` contains the logic which this authentication provider should be applied or not.

```java
public interface AuthenticationProvider {
	Authentication authenticate(Authentication authentication) throws AuthenticationException;
	boolean supports(Class<?> authentication);
}
```

For the `authenticate()` method, there are three options:

1. If the request is authenticated, should return authenticated Authentication instance
2. If the request is not authenticated, throw AuthenticationException
3. If the Authentication isn't supported by this provider, then return null, in another words AuthenticationProvider will say the AuthenticationManager: "**Hey AuthenticationManager please try to use another Providers, I am not responsible for this**"

For the `supports()` method:

- This method is called by the AuthenticationManager, and this method should check the authentication type. For instance: `return UsernamePasswordAuthenticationToken.class.equals(authentication);` means that authentication should be processed when authentication type is UsernamePasswordAuthenticationToken (httpBasic uses this authentication)

Here is the `MyCustomAuthenticationProvider`:

> Note: When you are returning fully authenticated instance, return an instance that implements Authentication interface and `Authenticate#isAuthenticated` **must return true**.
>
> For httpBasicAuthentication, instance will be `UsernamePasswordAuthenticationToken,`
>
> ```java
> public class UsernamePasswordAuthenticationToken extends AbstractAuthenticationToken {
> 	// for fully authentication instance, DO NOT USE THIS ONE !!!
> 	public UsernamePasswordAuthenticationToken(Object principal, Object credentials) {
> 		super(null);
> 		this.principal = principal;
> 		this.credentials = credentials;
> 		setAuthenticated(false);
> 	}
>
>   // for fully authentication instance, USE THIS ONE
> 	public UsernamePasswordAuthenticationToken(Object principal, Object credentials,
> 			Collection<? extends GrantedAuthority> authorities) {
> 		super(authorities);
> 		this.principal = principal;
> 		this.credentials = credentials;
> 		super.setAuthenticated(true);
> 	}
> }
> ```

```java
/**
 * This class includes the following logic:
 * If a user exists and password is correct, then login must be successful
 * otherwise login should fail
 */
@Component
public class MyCustomAuthenticationProvider implements AuthenticationProvider {
    @Autowired
    private UserDetailsService userDetailsService;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String username = authentication.getName();
        String password = String.valueOf(authentication.getCredentials());

        UserDetails userDetails = userDetailsService.loadUserByUsername(username);
        // In real case scenarios, userDetailsService should throw an error when user is not found
        // therefore there is no need for null check
        if (userDetails != null){
            if (passwordEncoder.matches(password, userDetails.getPassword())){
               UsernamePasswordAuthenticationToken authenticationToken =
                       new UsernamePasswordAuthenticationToken(username, password, userDetails.getAuthorities());
                return authenticationToken;
            }
        }
        throw new BadCredentialsException("Error!!");
    }

    /**
     * Because I am going to use HttpBasicAuthentication
     * and HttpBasicAuthentication uses UsernamePasswordAuthenticationToken
     * @param authenticationType
     * @return
     */
    @Override
    public boolean supports(Class<?> authenticationType) {
        return UsernamePasswordAuthenticationToken.class.equals(authenticationType);
    }
}
```

The last step is to add the custom provider to the configuration:

```java
@Configuration
public class ProjectBeanConfiguration  extends WebSecurityConfigurerAdapter {
    @Autowired
    private MyCustomAuthenticationProvider myCustomAuthenticationProvider;

   // ...
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.authenticationProvider(myCustomAuthenticationProvider);
    }
}
```

After starting the application, run the following curl command: (You can also add the debug point to the custom authentication provider)

```bash
[mehmetozanguven@localhost ~]$ curl --user dummy_user:1234 -X GET http://localhost:8080/hello

hello
```

I will continue with the next one ...
