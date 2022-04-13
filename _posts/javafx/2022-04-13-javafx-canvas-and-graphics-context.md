---
layout: post
title: "JavaFx Canvas and GraphicsContext"
date: 2022-04-13 12:45:31 +0530
categories: "javafx"
author: "mehmetozanguven"
---

The screen of a computer is a grid of little squares called **pixels**. The color of each pixel can be set individually, and drawing on the screen just means setting the colors of individual pixels.

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

Every visible GUI component is drawn by coloring pixels, and every component has a coordinate system that maps (x, y) coordinates to points within the component.

Most components draw themselves, but there is one JavaFX component on which we can draw anything we want by calling correct methods. Such **drawing surface** components are called **Canvas**

> A canvas is a node.

A canvas appears on the screen as a rectangle which made up of pixels. A position in the rectangle is specified by a pair of coordinates (x, y). Coordinate has the following properties:

- The upper left corner has coordinates (0,0)
- The x coordinate increases from left to right
- The y coordinate increases from top to bottom.

<img src="/assets/javafx/canvas_and_graphics_context/javafx_canvas_corrdinate.png" alt="javafx_canvas_coordinate.png" />

The width and height of the canvas can be specified when creating the canvas object. For instance:

```java
// create 20 by 12 canvas
Canvas canvas = new Canvas(20, 12);
```

When canvas object is created, it has color with all RGBA components set to 0.

In order to draw on a canvas, we need an object of type **GraphicsContext**. Every Canvas has a `GraphicsContext` and different `GraphicsContext` draw on different `Canvases` .

We can get the graphics context for a Canvas by calling `canvas.getGraphicsContext2D()` .

> Note that for the same canvas, this method: `canvas.getGraphicsContext2D()` always returns the same graphic context.

Some methods about the graphics context:

- `g.strokeRect(x, y, w, h)` & `g.fillRect(x, y, w, h)` - Draw a rectangle with top left corner at (x, y) with width w and with height h.
- `g.clearRect(x,y,w,h)` - Fill the same rectangle with a fully transparent color, so that whatever lies behind the rectangle will be visible through the canvas.
- `g.strokeOval(x, y, w, h)` & `g.fillOval(x, y, w, h)` - Draw an oval that just **fits inside the rectangle with top left corner at (x, y) with width w and with height h**
- `g.strokeLine(x1, y1, x2, y2)` - Draw a line from (x1, y1) and (x2, y2)

GraphicContext has a number of properties that affect drawing and each property has a setter method such as `g.setFill(paint)` . **However changing the value of a property does not affect anything that has already been draw, the change only applies to things drawn in the future**

> You may ask what is the stroke (or especially stroking shape) and fill (or especially filling the shape)
>
> Stroke - is a process of drawing a shape's outline applying stroke width, line style and color attribute. Here is the example of the dashed style of rectangle:
>
> <img src="/assets/javafx/canvas_and_graphics_context/stroke_dashed_rectangle.png" alt="stroke_dashed_rectangle.png" />
>
> Filling - is a process of painting the shape’s interior with solid color or a color gradient, or a texture pattern. Here is the example:
>
> <img src="/assets/javafx/canvas_and_graphics_context/fill_shape.png" alt="fill_shape" />

It is also possible to draw an image onto a canvas. Here are the several methods:

- `g.drawImage(image, x, y)` - Draws the image with its upper left corner at (x,y) using the actual size of the image.
- `g.drawImage(image, x, y, w, h)` - Draws the image in the rectangle with upper left corner at (x,y) with with w and height h. The image will be stretched or shrunk to fit in the rectangle.
- `g.drawImage(image, sx, sy, sw, sh, dx, dy, dh, dw)` - Draws the content of a specified source rectangle in the image to a specified destination rectangle on the canvas. This method lets we draw just part of an image. The source rectangle has upper left corner at (sx, sy), width sw and height sh. The last four parameters specify the destination rectangle.

## Conclusion

- The screen of a computer is a grid of little squares called **pixels**.
  - The color of each pixel can be set individually,
- Most components draw themselves, but there is one JavaFx component on which we can draw anything we want by calling correct methods. Such **drawing surface** components are called **Canvas**
- A canvas is actually a node.
- A canvas appears on the screen as a rectangle which made up of pixels.
- In order to draw on a canvas, we need an object of type **GraphicsContext**. Every Canvas has a `GraphicsContext` and different `GraphicsContext` draw on different `Canvases`.
- It is also possible to draw an image onto a canvas.
