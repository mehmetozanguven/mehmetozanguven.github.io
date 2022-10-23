---
layout: post
title: How to resolve -- Invalid Cors Request In Spring Boot
date: 2021-05-13 14:45:31 +0530
categories: "how-to"
author: "mehmetozanguven"
---

Recently I have encountered this error. And I can not send any request even I have setup `corsConfiguration.setAllowedOrigins("*")` . Response was only saying => `Invalid CORS request`

<br />
<br />

---

Checkout my udemy course **[https://www.udemy.com/course/learning-spring-security-fundamentals/](https://www.udemy.com/course/learning-spring-security-fundamentals/?referralCode=04AD7C0E89610E136C69)** if you really want to know fundemantals of the Spring Security and more ...

---

<br />
<br />

Solution is that if you setup your cors configuration through CorsFilter you should set the allowed methods.

Here is the example:

```java
@Component
@EnableWebSecurity
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    @Bean
    public CorsFilter corsFilter() {
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        CorsConfiguration corsConfiguration = new CorsConfiguration();
        corsConfiguration.setAllowedHeaders(List.of("*"));
        corsConfiguration.setAllowedOrigins(Arrays.asList("*"));
        corsConfiguration.setAllowedMethods(Arrays.asList("*")); // add this line with appropriate methods for your case
        corsConfiguration.setMaxAge(Duration.ofMinutes(10));
        source.registerCorsConfiguration("/**", corsConfiguration);
        return new CorsFilter(source);
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable();
        http.authorizeRequests().anyRequest().permitAll();
        http.cors();
    }
}
```

## Where Invalid CORS request was being fired ?

In the `DefaultCorsProcessor.handleInternal(...)` method:

```java
	protected boolean handleInternal(ServerHttpRequest request, ServerHttpResponse response,
			CorsConfiguration config, boolean preFlightRequest) throws IOException {

		// ...
		HttpMethod requestMethod = getMethodToUse(request, preFlightRequest);
		List<HttpMethod> allowMethods = checkMethods(config, requestMethod);
		if (allowMethods == null) {
			logger.debug("Reject: HTTP '" + requestMethod + "' is not allowed");
			rejectRequest(response);
			return false;
		}

        // ...

		responseHeaders.setAccessControlAllowOrigin(allowOrigin);

		// ...
	}

protected void rejectRequest(ServerHttpResponse response) throws IOException {
	response.setStatusCode(HttpStatus.FORBIDDEN);
	response.getBody().write("Invalid CORS request".getBytes(StandardCharsets.UTF_8));
	response.flush();
}
```
