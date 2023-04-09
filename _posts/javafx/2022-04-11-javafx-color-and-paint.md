---
layout: post
title: "JavaFX Color and Paint"
date: 2022-04-11 12:45:31 +0530
categories: "javafx"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/javafx/color-and-paint/"
---

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

## Color

Computer color uses an **RGB color system** which means that color on the a computer screen is specified by three numbers called **color components**, giving the level of **red, green and blue** in the color

In JavaFX, color is represented by type of `Color` class (from package `javafx.scene.paint`). And each color component is a double value in the range 0.0 and 1.0

Color object also has fourth component in the range 0.0 to 1.0, referred as the **alpha color component** used to represent the transparency or opaqueness of the color.

To create color object:

```java
Color color = new Color(r, g, b, a);
// Or use factory method
Color color = Color.color( r, g, b, a );
```

> **You should use the factory method. In that way you can use the same objects for the same color (there is no reason to create new objects for the same color).**
>
> Also color objects are immutable, which means there is no way to change a color after it has been constructed.

If your computer uses 32 bit color, it means that the color of each pixel is actually represented using just 8 bits for each of the four color components. Eight bits can take integers from 0 to 255. So computer colors have traditionally been specified using integer color components in the range 0 to 255. JavaFX has a static method to create a color within the range 0 to 255

```java
Color.rgb(r, g, b);
Color.rgb(r, g, b, a);// where r, g, and b are integers in the range 0 to 255.
// alpha color component(a) is a double in the range 0.0 to 1.0
```

## HSB Color system

An alternative to RGB is the **HSB color system**

IN HSB, a color is represented by three numbers called the **hue, saturation and brightness**.

- The hue = color of the rainbow, represented by a double value from 0.0 to 360.0
- The brightness is pretty much what it sounds like. Represented by a double value in the range 0.0 to 1.0
- The saturation is like mixing white or gray paint into the pure color. Represented by a double value in the range 0.0 to 1.0

To create a color from HSB

```java
Color hsbColor = Color.hsb(h, s, b);
Color hsbColor = Color.hsb(h, s, b, a);
```

## Paint

Color is a subclass of `Paint` which is a base class for a color or gradients used to fill shapes and backgrounds when rendering the scene graph. We can talk about Paint later on ...

## Font

A **font** represents a size and style of text.

In JavaFX we have object for the fonts called `Font`.

We can create font object by using its static factory methods.

If system can't find the given font name, it will try to match the best one it finds. If you want to match with the system font size, you can pass as null for the font name

In general we can create font in JavaFX:

```java
Font fontCreation = Font.font(family, weight, posture, size);
/*
- Family is a String which specifies a font family such as "Times New Roman"
- weight is the enumerated type such as FontWeight.BOLD, FontWeight.NORMAL
- posture is also the enumerated type such as FontPosture.ITALIC, FontPosture.REGULAR
*/

Font font1 = Font.font(23);
Font font2 = Font.font("Times New Roman", FontWeight.BOLD, 14);
Font font3 = Font.font(null, FontWeight.BOLD, FontPosture.REGULAR, 44);
```

## Image

We can use `Image` object to show "image" in the JavaFX program. For instance, we can load image from the resources folder.

```java
Image test = new Image("test.png");
```

The image can be displayed in a `GraphicsContext`

## Conclusion

- We looked at the basic classes for the JavaFX such as, Color, Font, Image
- Computer color uses an **RGB color system**
  - It means that color on the a computer screen is specified by three numbers called **color components**, giving the level of **red, green and blue (RGB)** in the color
  - An alternative to RGB is the **HSB color system**
- Color object also has fourth component in the range 0.0 to 1.0, referred as the **alpha color component**
  - It is used to represent the transparency or opaqueness of the color.
- We should use the factory method. In that way we can use the same objects for the same color (there is no reason to create new objects for the same color).
- A **font** represents a size and style of text.
  - We can create font object by using its static factory methods.
- We can use `Image` object to show "image" in the JavaFX program.
  - The image can be displayed in a `GraphicsContext`
