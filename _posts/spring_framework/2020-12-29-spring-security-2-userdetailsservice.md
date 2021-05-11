---
layout: post
title: Spring Security -- 2) UserDetailsService"
date: 2020-12-29 16:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

In this post, I am going to use real database to check the user against the request(s).

Topics are:

- [**Github Link**](#github_link)
- [**Default Project Setup**](#default_setup)
- [**Architecture**](#architecture)
- [**Creating UserDetailsService**](#creating_userDetailsService)

## Github Link <a name="github_link"></a>

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/spring-security-examples/tree/master/spring-security-with-database)

## Default Setup for Project <a name="default_setup"></a>

I am going to use Postgresql, and I created new spring boot application with these dependencies:

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

And there is also one endpoint in the project:

```java
package com.mehmetozanguven.springsecuritywithdatabase.controller;

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

Do not forget to add database configuration in the application.properties file:

```properties
# Spring DATASOURCE for local postgresql
spring.datasource.url=jdbc:postgresql://localhost:5432/testdatabase
spring.datasource.username=postgres
spring.datasource.password=1234

# For localdb, it would be good to re-create all-tables
spring.jpa.hibernate.ddl-auto=update
```

Please create the table and initialize with predefined data:

> In our example, because I use the NoOpPasswordEncoder, I do not need to find hashcode for the dummy inserted user.
>
> **Do not use NoOpPasswordEncoder in the production**

```bash
[mehmetozanguven@localhost ~]$ sudo -iu postgres
[postgres@localhost ~]$ psql
postgres=# \c testdatabase
You are now connected to database "testdatabase" as user "postgres".
testdatabase=# CREATE TABLE IF NOT EXISTS test_users (id serial PRIMARY KEY, username VARCHAR(50), password VARCHAR(50));
testdatabase=# INSERT INTO test_users (username,password) VALUES ('dummy_user','1234');
```

DTO class:

```java
package com.mehmetozanguven.springsecuritywithdatabase.entity;

@Entity
@Table(name = "test_users")
public class UserDTO {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column(name = "username")
    private String username;

    @Column(name = "password")
    private String password;

   // getters and setters
}
```

Repository to connect the database:

```java
package com.mehmetozanguven.springsecuritywithdatabase.repository;

@Repository
public interface UserRepository extends JpaRepository<UserDTO, Long> {

    Optional<UserDTO> findUserByUsername(String username);
}
```

And Spring Security needs to work with `UserDetails` object, therefore I should somehow convert to the UserDTO to the `UserDetails` object. I have created a class for that:

```java
package com.mehmetozanguven.springsecuritywithdatabase.services;

import com.mehmetozanguven.springsecuritywithdatabase.entity.UserDTO;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

public class SecureUser  implements UserDetails {

    private final UserDTO userDTO;

    public SecureUser(UserDTO userDTO) {
        this.userDTO = userDTO;
    }

    /**
     * Because I will not look at the authority part for now,
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

    /**
     * Because we do not have any logic for this method
     * just return true to pass.
     */
    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    /**
     * Because we do not have any logic for this method
     * just return true to pass.
     */
    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    /**
     * Because we do not have any logic for this method
     * just return true to pass.
     */
    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    /**
     * Because we do not have any logic for this method
     * just return true to pass.
     */
    @Override
    public boolean isEnabled() {
        return true;
    }
}
```

## Architecture <a name="architecture"></a>

Before diving into details, that would be good to have picture describes the overall architecture:

<img src="/assets/spring/security/userdetailsservice/jdbcuserdetailsflow.png" alt="jdbcuserdetailsflow" />

Because I am going to find out whether there is a user for that request in our database, let's create userDetailsService namely `PostgresqlUserDetailsService`:

## Creating UserDetailsService <a name="creating_userDetailsService"></a>

```java
package com.mehmetozanguven.springsecuritywithdatabase.services;

public class PostgresqlUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username){
        Optional<UserDTO> userDTOOptional = userRepository.findUserByUsername(username);
        UserDTO userInDb = userDTOOptional.orElseThrow(() -> new UsernameNotFoundException("Not Found in DB"));
        return new SecureUser(userInDb);
    }
}
```

Here is the configuration (this is not a security configuration that you would expect, in this configuration i am just creating my own beans for `userdetailsService` and `passwordEncoder` therefore spring will use my beans instead of the default ones):

```java
package com.mehmetozanguven.springsecuritywithdatabase.config;

@Configuration
public class ProjectBeanConfiguration {
    @Bean
    public UserDetailsService userDetailsService(){
       return new PostgresqlUserDetailsService();
    }

    @Bean
    public PasswordEncoder passwordEncoder(){
        return NoOpPasswordEncoder.getInstance();
    }
}
```

After running the project, here is the curl command to access the endpoint:

```bash
[mehmetozanguven@localhost ~]$ curl --user dummy_user:1234 -X GET  http://localhost:8080/hello

hello
```

I will continue with the next one...
