---
layout: post
title: "JavaFX and CSS (Final)"
date: 2022-06-12 20:00:31 +0530
categories: "javafx"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/javafx/javafx-final/"
---

We can use CSS(Cascading Style Sheets) to change visual appearance of our JavaFX program.

> Anything that can be done with CSS can also be done with Java
>
> Be careful not every feature in the CSS can be used in the JavaFX. For instance JavaFX CSS does not support CSS layout properties such as float, position, overflow, and width.
>
> You can learn more about JavaFX CSS in here [https://docs.oracle.com/javafx/2/api/javafx/scene/doc-files/cssref.html](https://docs.oracle.com/javafx/2/api/javafx/scene/doc-files/cssref.html)

All JavaFX CSS properties begin with **-fx** to distiguish them from regular CSS properties.

We can apply a style to a component using the method `setStyle(string)`. We can add many css properties separating by commas:

```java
message.setStyle(
"-fx-padding: 5px; -fx-border-color: black; -fx-border-width: 1px" );
```

We can also add css to our JavaFX program by specifying the **.css** file.

Here is the example for styling all Buttons:

```css
Button {
  -fx-font: italic 16pt "Times New Roman";
  -fx-text-fill: green;
}
```

Then we can load this css file from the resources:

```java
scene.getStylesheets().add("custom_style.css");
// Scene can have several style sheets
// css files can also be added to individual containers
```
