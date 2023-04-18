---
layout: post
title: How to use cache in Spring Boot  applications without annotation(s)
date: 2022-01-16 15:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/spring-boot/spring-boot-application-without-cache-annotations/"
---

Almost every blog(s) on the Internet for caching operation in the Spring Boot are using `@Cacheable`, `@CacheEvit`, `@CachePut` vs... In this blog we are going to implement cache without using annotations instead we will create&use service class to get records from cache.

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

## Github link

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/spring-boot-examples/tree/master/spring-cache-without-annotation)

## Cache Provider

Spring boot supports several cache providers. If it finds a cache provider on the classpath, then it tries to find default configuration for that provider. Otherwise it just configures the simple cache provider which is just a `ConcurrentHashMap`

In this example I am going to use hazelcast as a cache provider. Create Spring boot projects with the following dependencies:

> I don't want to bother you with postgresql setup, that's why I am going to use h2 in-memory store.

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
    <!-- Hazelcast for caching -->
    <dependency>
        <groupId>com.hazelcast</groupId>
        <artifactId>hazelcast</artifactId>
    </dependency>
    <dependency>
        <groupId>com.hazelcast</groupId>
        <artifactId>hazelcast-spring</artifactId>
    </dependency>

    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

## Creating Schema and Inserting Data on initialization

We need to initialize the database and we also need to add some records inside table. We can do that by creating two files in `src/main/resources/` folder called :

- **`schema.sql`**: To initialize the schema e.g create tables
- **`data.sql`**: To insert rows

These two files (schema & data) will be read by the Spring boot automatically.

Here is the schema.sql:

```sql
DROP TABLE IF EXISTS users;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(250),
    last_name VARCHAR(250),
    email VARCHAR(250)
);
```

Here is the data.sql:

```sql
INSERT INTO users (first_name, last_name, email) VALUES ('Jerry', 'Gupta', 'jerrygupta@email.com');
INSERT INTO users (first_name, last_name, email) VALUES ('Ronaldo', 'Sanchez', 'ronaldo@email.com');
INSERT INTO users (first_name, last_name, email) VALUES ('Amy', 'America', 'amy@gmail.com');
```

## Create basic project setup without caching

We will have basic setup to find user by id, by name, by email etc...

### Repository

`UserRepository`is nothing but just extend the `JpaRepository`:

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    @Query("FROM User u WHERE u.firstName = ?1")
    Optional<User> findByFirstName(String firstName);
}
```

### Service

Any user service has to implement following methods:

```java
public interface UserService {
    User findById(long id);
    User findByFirstName(String firstName);
    void changeFirstName(long id, String firstName);
    List<User> getAllUsers();
}
```

### Controller

This is so basic controller to call every service methods:

```java
@RestController
@RequestMapping(value = "/api")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/findById")
    public ResponseEntity<User> findById(@RequestParam(value = "id") long userId) {
        User user = userService.findById(userId);
        return ResponseEntity.ok(user);
    }

    @GetMapping("/findByUserName")
    public ResponseEntity<User> findByFirstName(@RequestParam(value = "name") String firstName) {
        User user = userService.findByFirstName(firstName);
        return ResponseEntity.ok(user);
    }

    @PostMapping("/updateFirstName")
    public ResponseEntity<String> updateFirstName(@RequestParam(value = "id") long userId, @RequestParam(value = "name") String newFirstName) {
        userService.changeFirstName(userId, newFirstName);
        return ResponseEntity.ok("Success");
    }

    @GetMapping("/findAll")
    public ResponseEntity<List<User>> findAllUsers() {
        return ResponseEntity.ok(userService.getAllUsers());
    }
}
```

## Testing the application

Get all users:

```bash
$ curl -X GET http://localhost:8080/api/findAll
```

```json
[
  {
    "id": 1,
    "firstName": "Jerry",
    "lastName": "Gupta",
    "email": "jerrygupta@email.com"
  },
  {
    "id": 2,
    "firstName": "Ronaldo",
    "lastName": "Sanchez",
    "email": "ronaldo@email.com"
  },
  {
    "id": 3,
    "firstName": "Amy",
    "lastName": "America",
    "email": "amy@gmail.com"
  }
]
```

We can find user also by id:

```bash
$ curl -X GET http://localhost:8080/api/findById?id=2
```

```json
{
  "id": 2,
  "firstName": "Ronaldo",
  "lastName": "Sanchez",
  "email": "ronaldo@email.com"
}
```

<br />

Now let's add cache to our application

## Add Hazelcast Cache to existing application

To add hazelcast as a cache provider to our spring boot application we need to do the following things:

- Add **spring-boot-starter-cache** as a dependency (we already did this !!)

- Add `@EnableCaching` annotation your configuration class or main class

- Add hazelcast configuration (either yaml or create a Bean) (we are going to look at the bean configuration)

> Don't forget to implement Serializable interface for cachable object(s)

- Finally you are ready to use `@Cachable`and more annotations

### Add `@EnableCaching`annotation

Let's create a configuration class called `HazelcastCacheConfiguration` and annotated with `@EnableCaching`

```java
@Configuration
@EnableCaching
public class HazelcastCacheConfiguration {
}
```

### Create a hazelcast configuration bean

```java
@Configuration
@EnableCaching
public class HazelcastCacheConfiguration {
     private static final String HAZELCAST_INSTANCE_NAME = "my-hazelcast-instance";


    @Bean
    public Config hazelCastConfig(){
        EvictionConfig evictionConfig = new EvictionConfig();
        evictionConfig.setEvictionPolicy(EvictionPolicy.LRU);
        evictionConfig.setMaxSizePolicy(MaxSizePolicy.USED_HEAP_SIZE);

        MapConfig cacheConfig = new MapConfig();
        cacheConfig.setName("my-cache-name");
        cacheConfig.setEvictionConfig(evictionConfig);
        cacheConfig.setTimeToLiveSeconds(3000);

        Config config = new Config();
        config.setInstanceName(HAZELCAST_INSTANCE_NAME)
                .addMapConfig(cacheConfig);
        return config;
    }
}
```

I believe code is self-explanatory. Record(s) in the cache will be deleted by Least Recently Used policy. And each record(s) in the cache can stay at maximum 3000seconds(After 3000 seconds they will be automayically removed from the cache)

Now our cache configuration is ready. After you started the application, you will see Hazelcast members in the logs:

```text
2022-01-16 14:43:58.949  INFO 78957 --- [           main] com.hazelcast.system                     : [192.168.1.16]:5701 [dev] [4.1.6] Hazelcast 4.1.6 (20211026 - b77402f) starting at [192.168.1.16]:5701
...
...
Members {size:1, ver:1} [
	Member [192.168.1.16]:5701 - 639faa4a-0ddd-4a80-a4a0-b043fb611316 this
]
...
...
```

## Create service class to look up records in the cache

Because we want to access cache programmatically, we can do that by creating another service. Create a class called `MyHazelcastServiceImpl`

```java

import com.hazelcast.core.HazelcastInstance;

@Service
public class UserHazelcastServiceImpl  {
    private final HazelcastInstance hazelcastInstance;

    public UserHazelcastServiceImpl(@Qualifier("hazelcastInstance") HazelcastInstance hazelcastInstance) {
        this.hazelcastInstance = hazelcastInstance;
    }
}
```

Spring boot will automatically insert correct implementation for the hazelcast instance. The left is to just define appropriate methods.

### Create a cache method for user look up via id field

```java
@Service
public class UserHazelcastServiceImpl {
    private final HazelcastInstance hazelcastInstance;

    public UserHazelcastServiceImpl(@Qualifier("hazelcastInstance") HazelcastInstance hazelcastInstance) {
        this.hazelcastInstance = hazelcastInstance;
    }

    public User findByIdInCache(long userId) {
        Map<Long, User> userCache = hazelcastInstance.getMap("my-cache-name");
        return userCache.get(userId);
    }

    public void putUserInTheCache(User user) {
        Map<Long, User> userCache = hazelcastInstance.getMap("my-cache-name");
        userCache.put(user.getId(), user);
    }
}
```

Finally before sending request to the repository we will first lookup the cache:

```java

@Transactional
public class UserServiceImpl implements UserService {
    private static final Logger logger = LoggerFactory.getLogger(UserServiceImpl.class);
    private final UserRepository userRepository;
    private final UserHazelcastServiceImpl userHazelcastService;

    public UserServiceImpl(UserRepository userRepository,
                           UserHazelcastServiceImpl userHazelcastService) {
        this.userRepository = userRepository;
        this.userHazelcastService = userHazelcastService;
    }


    @Override
    public User findById(long id) {
        User user = userHazelcastService.findByIdInCache(id);
        if (user != null) {
            logger.info("Found inside the cache. User: {}", user);
            return user;
        }
        logger.info("User did not found in the cache. DB lookup...");
        User dbInUser = userRepository.findById(id).orElseThrow(() -> new RuntimeException("Invalid request"));
        userHazelcastService.putUserInTheCache(dbInUser);
        return dbInUser;
    }
}
```

> Note: Maybe using proxy pattern would be more readable solution rather than updating findById method in the UserServiceImpl.
>
> To implement proxy pattern:
>
> - UserHazelcastServiceImpl("ProxyUserService") must implement UserService
>
> - You must inject UserServiceImpl("OriginalUserService") to UserHazelcastServiceImpl class
>
> - Then for each method in the UserService, you will call the implementations from the UserHazelcastServiceImpl. If user is found in the cache, then there is no need to call UserServiceImpl, otherwise just call the implementations from the UserService and add the response to the cache and return the result.

After running the application, just send the following request with twice:

```bash
 curl -X GET http://localhost:8080/api/findById?id=2
```

In the application logs:

```text
2022-01-16 15:04:07.995  INFO 79649 --- [nio-8080-exec-1] c.m.s.service.UserServiceImpl            : User did not found in the cache. DB lookup...

2022-01-16 15:04:14.253  INFO 79649 --- [nio-8080-exec-2] c.m.s.service.UserServiceImpl            : Found inside the cache. User: com.mehmetozanguven.springcachewithoutannotation.model.User@175e94dd

```

As you can see, we received response from the cache for the second request.

## Conclusion

In this blog, we tried to answer the question "how to use cache without annotation". Basically to enable hazelcast cache in the spring boot application, you should (at least) do the following:

- Add spring-boot-starter-cache as a dependency

- Add the following hazelcast dependency to work with it:

```xml
<!-- Hazelcast for caching -->
    <dependency>
        <groupId>com.hazelcast</groupId>
        <artifactId>hazelcast</artifactId>
    </dependency>
    <dependency>
        <groupId>com.hazelcast</groupId>
        <artifactId>hazelcast-spring</artifactId>
    </dependency>
```

- Then add `@EnableCaching`annotation your main class or configuration class

- After all create hazelcast configuration either yaml file or bean style.

To lookup records in the cache without annotation, you must create an service class and inject the `HazelcastInstance` bean.

Finally don't forget that cache name in the `hazelcastInstance.getMap("my-cache-name");`must match to the cache name in the configuration class.
