---
layout: post
title: How to read properties file only once in the Spring Boot application
date: 2021-11-26 13:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

Do you want to know how to read specific properties file according to the environment variable in Spring boot? Let's explore how to read properties file in a different way that Spring already provides.

You probably know that Spring Framework provides convenient way to read property from correspond properties file(s). But Spring's solution only works in the Spring context. Sometimes you may need to get property value outside of the Spring context. For instance you may need to load properties file inside an utility class.

Let's say we have property file called **user_auth.properties** both for local and test environments.

Here is the **user-auth.properties** file for local environment:

```properties
header.name=local
header.value=1234
```

**user-auth.properties** file for prod environment:

```properties
header.name=prod
header.value=12345
```

Here is the resource direction:

<img src="/assets/spring/howTo/read_properties_file_once/file_structure.png" alt="resource file structure" />

We can use `getResourceAsStream()` method to load properties files. But we should first determine the environment in which our spring boot application will run.

We can determine environment using `spring.profile.active` property value which is setting by us when we run our application. For instance:

```bash
$ java -jar -Dspring.profiles.active=prod spring-boot.jar
```

Because we will use `spring.profiles.active`, our spring application will also apply prod environment values.

**To load properties file only once, we can apply Singleton pattern on the appropriate class(es)**.

First we cat set the environment via class `EnvironmentProperties`. If there is no `spring.profiles.active`property then we should set it as local environment (this behavior actually depends on your case, you may also throw exception at this time).

```java
public class EnvironmentProperties {
    public enum Environment {
        LOCAL("local"), PROD("prod");
        public final String value;

        Environment(String value) {
            this.value = value;
        }
    }

    interface Constants {
        String PROD_FILE_PATH_PREFIX = "prod";
        String LOCAL_FILE_PATH_PREFIX = "local";
    }

    private static final String ENVIRONMENT_PROPERTY = "spring.profiles.active";

    private static EnvironmentProperties singletonInstance;

    private Environment environment;

    public static synchronized EnvironmentProperties getInstance() {
        if (singletonInstance == null) {
            singletonInstance = new EnvironmentProperties();
        }
        return singletonInstance;
    }

    private EnvironmentProperties() {
        logger.info("Loading environment properties");
        setEnvironment();
        logger.info("Loaded environment properties");
    }

    private void setEnvironment() {
        String environment = System.getProperty(ENVIRONMENT_PROPERTY);
        if (StringUtils.isEmpty(environment)) {
            logger.info("Environment value is empty. Defaulting local");
            this.environment = Environment.LOCAL;

        } else {
            logger.info("Environment value is: {}", environment);
            if (Environment.LOCAL.value.equals(environment)) {
                this.environment = Environment.LOCAL;
            } else if (Environment.PROD.value.equals(environment)) {
                this.environment = Environment.PROD;
            } else {
                logger.error("Invalid environment value");
                throw new RuntimeException("Invalid environment value");
            }
        }
    }

    public Environment getEnvironment() {
        return environment;
    }

    public String getPathPrefix() {
        if (Environment.PROD.equals(environment)) {
            return Constants.PROD_FILE_PATH_PREFIX;
        } else {
            return Constants.LOCAL_FILE_PATH_PREFIX;
        }
    }
}
```

After that we can use the `EnvironmentProperties` to load appropriate **user_auth.properties**.

If environment value is`Environment.LOCAL`, then `SecureUserProperties` will load the property in the path: `local/user_auth.properties`.

If environment value is`Environment.PROD`, then `SecureUserProperties` will load the property in the path: `prod/user_auth.properties`.

```java
public class SecureUserProperties {

    public interface PropertyKeys {
        String HEADER_NAME = "header.name";
        String HEADER_VALUE = "header.value";
    }

    private static final String PROPERTY_NAME = "user_auth.properties";
    private static final EnvironmentProperties environmentProperties = EnvironmentProperties.getInstance();

    private static SecureUserProperties singletonInstance;

    private Properties userAuthProperties;


    public static synchronized SecureUserProperties getInstance() {
        if (singletonInstance == null) {
            singletonInstance = new SecureUserProperties();
        }
        return singletonInstance;
    }

    private SecureUserProperties()  {
        logger.info("Loading properties file: {}", PROPERTY_NAME);
        try{
            loadUserAuthProperties();
            logger.info("Loaded properties file: {}", PROPERTY_NAME);
        } catch (Exception ex) {
            logger.error("Error while loading the properties: {}", PROPERTY_NAME, ex);
            throw new RuntimeException("Error while loading the properties");
        }
    }

    private void loadUserAuthProperties() throws IOException {
        String pathPrefix = environmentProperties.getPathPrefix();
        userAuthProperties = new Properties();
        InputStream inputStream = getClass().getClassLoader().getResourceAsStream(pathPrefix + "/" + PROPERTY_NAME);
        userAuthProperties.load(inputStream);
    }

    public String getHeaderName() {
        return userAuthProperties.getProperty(PropertyKeys.HEADER_NAME);
    }

    public String getHeaderValue() {
        return userAuthProperties.getProperty(PropertyKeys.HEADER_VALUE);
    }
}
```

To load property when spring boot application is starting to run, we can use `@PostConstruct` annotation:

```java
@SpringBootApplication
public class MyApplication {

	public static void main(String[] args) {
		SpringApplication.run(MyApplication.class, args);
	}

	@PostConstruct
	public void init(){
		EnvironmentProperties environmentProperties = EnvironmentProperties.getInstance();
		SecureUserProperties secureUserProperties = SecureUserProperties.getInstance();
	}
}
```

If I run the project from the IDEA, console output will be:

```wiki
22:09:59.242 [main] INFO  c.m.r.p.EnvironmentProperties - Loading environment properties
22:09:59.251 [main] INFO  c.m.r.p.EnvironmentProperties - Environment value is empty. Defaulting local
22:09:59.251 [main] INFO  c.m.r.p.EnvironmentProperties - Loaded environment properties

22:09:59.251 [main] INFO  c.m.r.p.SecureUserProperties - Loading properties file: user_auth.properties
22:09:59.252 [main] INFO  c.m.r.p.SecureUserProperties - Loaded properties file: user_auth.properties
```

That's it, using Singleton pattern, we can access our properties anywhere in the codebase.
