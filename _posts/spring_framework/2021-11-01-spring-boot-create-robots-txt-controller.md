---
layout: post
title: How to create robots.txt controller in Spring Boot
date: 2021-11-01 13:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/spring-boot/spring-boot-create-robots-txt-controller/"
---

Let's learn how to create robots.txt file for your spring boot or spring mvc project. Having robots.txt file tells search engine crawlers, such as google, which URLs the crawler can access on your website.

Without diving into further, let's assume that you have spring boot project. You can create `RobotsTxtController` class to return robots.txt file:

```java
@Controller
public class RobotsTxtController {
    @GetMapping(value = "/robots.txt", produces = MediaType.TEXT_PLAIN_VALUE)
    @ResponseBody
    public String getRobotsTxt() {
        return "User-agent: *\n" +
                "Allow: /\n";
    }
}
```

- With `@ResponseBody`, we are binding method return value to the web response body. In other words, we are just saying, don't return view from this endpoint.
- With `String TEXT_PLAIN_VALUE = "text/plain";`, we are saying that we will return text file.
- Returned value says that, "all crawlers can crawl any url on my website"

After you run the application in the port 8080, send request to this endpoint: `http://localhost:8080/robots.txt`
