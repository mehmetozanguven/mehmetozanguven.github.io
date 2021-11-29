---
layout: post
title: How to create sitemap.xml endpoint in Spring Boot
date: 2021-11-05 13:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

Let's quickly learn how to generate sitemap.xml endpoint for your spring boot project. We will create a sitemap controller to handle sitemap.xml requests.

> A _sitemap_ is a way of organizing a website, identifying the URLs and the data under each section.

Let's assume that you have spring boot or spring mvc application with the following domain and urls:

- Domain: `https://myblog.com`
- urls: `"/", "/projects", "/about", "/contactus"`

To create sitemap endpoint, you need to create new controller `SitemapController` with the following implementation:

```java
@Controller
public class SitemapController {
    private List<String> URLS = List.of("/", "/projects", "/about", "/contactus");
    private String DOMAIN = "https://myblog.com";

    @GetMapping(value = "/sitemap.xml")
    @ResponseBody
    public XmlUrlSet main() {
        XmlUrlSet xmlUrlSet = new XmlUrlSet();
        for (String eachLink : URLS) {
            create(xmlUrlSet, eachLink, XmlUrl.Priority.HIGH);
        }
        return xmlUrlSet;
    }

    private void create(XmlUrlSet xmlUrlSet, String link, XmlUrl.Priority priority) {
       xmlUrlSet.addUrl(new XmlUrl(DOMAIN + link, priority));
    }
}
```

After you run the project, send request to endpoint: `.../sitemap.xml`

```xml
<ns2:urlset>
	<url>
		<loc>https://myblog.com/</loc>
		<lastmod>2021-10-29</lastmod>
		<changefreq>daily</changefreq>
		<priority>1.0</priority>
	</url>
	<url>
		<loc>https://myblog.com/projects</loc>
		<lastmod>2021-10-29</lastmod>
		<changefreq>daily</changefreq>
		<priority>1.0</priority>
	</url>
	<url>
		<loc>https://myblog.com/about</loc>
		<lastmod>2021-10-29</lastmod>
		<changefreq>daily</changefreq>
		<priority>1.0</priority>
	</url>
	<url>
		<loc>https://myblog.com/contactus</loc>
		<lastmod>2021-10-29</lastmod>
		<changefreq>daily</changefreq>
		<priority>1.0</priority>
	</url>
</ns2:urlset>
```

Here are the `XmlUrl`and `XmlUrlSet`classes:

```java
@XmlAccessorType(value = XmlAccessType.NONE)
@XmlRootElement(name = "url")
public class XmlUrl {
    public enum Priority {
        HIGH("1.0"), MEDIUM("0.5");

        private String value;

        Priority(String value) {
            this.value = value;
        }

        public String getValue() {
            return value;
        }
    }

    @XmlElement
    private String loc;

    @XmlElement
    private String lastmod;

    @XmlElement
    private String changefreq = "daily";

    @XmlElement
    private String priority;

    public XmlUrl() {
        setLastmod();
    }

    public XmlUrl(String loc, Priority priority) {
        this.loc = loc;
        this.priority = priority.getValue();
        setLastmod();
    }

    private void setLastmod() {
        this.lastmod = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));
    }

    public String getLoc() {
        return loc;
    }

    public String getPriority() {
        return priority;
    }

    public String getChangefreq() {
        return changefreq;
    }

    public String getLastmod() {
        return lastmod;
    }
}
```

```java
@XmlAccessorType(value = XmlAccessType.NONE)
@XmlRootElement(name = "urlset", namespace = "http://www.sitemaps.org/schemas/sitemap/0.9")
public class XmlUrlSet {

    @XmlElements({@XmlElement(name = "url", type = XmlUrl.class)})
    private Collection<XmlUrl> xmlUrls = new ArrayList<XmlUrl>();

    public void addUrl(XmlUrl xmlUrl) {
        xmlUrls.add(xmlUrl);
    }

    public Collection<XmlUrl> getXmlUrls() {
        return xmlUrls;
    }
}
```
