---
layout: post
title: "How to add robots.txt and sitemap.xml into Quasar(VueJs) Framework"
date: 2022-03-27 12:45:31 +0530
categories: "others"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/how-to-add-robots-txt-and-sitemap-xml-into-quasar-framework/"
---

Adding robots.txt and/or sitemap.xml can be cumbersome for the modern javascript framework such as VueJs, React or Angular etc..

In this short blog, I will show you how to add robots.txt and sitemap.xml to your Quasar framework (based on VueJs)

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

## How to add robots.txt into Quasar project

- First create new file called **robots.txt** and put these contents:

```text
User-agent: *
Allow: /

Sitemap: https://yourWebsite.com/sitemap.xml
```

- Then put that file into the `/public` folder of your project.

After you run the Quasar project(or in the production build), you can access the robots.txt file by sending request to **https://yourWebsite.com/robots.txt**

## How to add sitemap.xml into Quasar project

- First create **sitemap.xml** file. You can use sitemap.xml generator sites. Or you can use the following template (replace the **loc**s with your router)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset
      xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.sitemaps.org/schemas/sitemap/0.9
            http://www.sitemaps.org/schemas/sitemap/0.9/sitemap.xsd">
<url>
  <loc>https://yourWebsite.com/</loc>
  <lastmod>2022-03-27T11:51:43+00:00</lastmod>
  <priority>1.00</priority>
</url>
<url>
  <loc>https://yourWebsite.com/page1</loc>
  <lastmod>2022-03-27T11:51:43+00:00</lastmod>
  <priority>0.80</priority>
</url>
<url>
  <loc>https://yourWebsite.com/page2</loc>
  <lastmod>2022-03-27T11:51:43+00:00</lastmod>
  <priority>0.80</priority>
</url>
<url>
  <loc>https://yourWebsite.com/page3</loc>
  <lastmod>2022-03-27T11:51:43+00:00</lastmod>
  <priority>0.80</priority>
</url>

</urlset>
```

- Then put that file into the `/public` folder of your project.

After you run the Quasar project(or in the production build), you can access the sitemap.xml file by sending request to **https://yourWebsite.com/sitemap.xml**

As a final note, you may also do some implementation to generate sitemap.xml according to the `src/router/index.js` in the build step.
