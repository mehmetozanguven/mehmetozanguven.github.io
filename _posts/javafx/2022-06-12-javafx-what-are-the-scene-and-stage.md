---
layout: post
title: "JavaFX - What are the Scene and Stage ?"
date: 2022-06-12 19:45:31 +0530
categories: "javafx"
author: "mehmetozanguven"
---

# Scene

- This is the one of the fundamental classes in JavaFX
- A **Scene** represents the content area of window (not include the window's border and title bar)
- A **Scene** servers as a holder for the root of the scene graph.
- All constructors of Scene need a root node which can not be null.
- A **Scene** has a width and height `(new Scene(root, width, height))`.
- In the typical case the root is a **Pane**.

  - The pane's size will be set to match the size of the scene
  - The pane will lay out its contents based on that size.
  - If the size of the scene is not specified, then the size of the scene will be set to the preferred size of the pane.

- A **Scene** can also have background color. But in general the scene's background is not seen, because background color will be covered the background color of the root node. If we want to see background color of the scene, we can set background color of the root node as transparent.

# Stage

- A **Stage** represents a window on the computer’s screen.
- **Any JavaFX Application has at least one stage, called the primary stage, which is created by the system and passed as a parameter to the application’s start() method.**
- We can create many window by creating many **Stage** objects.
- A **Stage** contains a scene, which fills its content area. It is also possible to show a stage that does not contain a scene, but its content area will just be a blank rectangle.
- A **Stage** has a title bar above the content. With that title bar we can be able do maximize and close the window.
  - The title bar is provided by the operating system, not by Java, and its style is set by the operating system. We can change text of title by calling `stage.setTitle(...)`
- By default a stage is resizable, we can disable this by calling `stage.setResizable(false)`
- Even a stage is resizable, we can also put the limits by calling `stage.setMinWidth(...) & stage.setMaxWidth(...)` and `stage.setMinHeight(...) & stage.setMaxHeight(...)`
- We can also change the position of a stage on the screen by calling `stage.setX(...) & stage.setY(...)`. The x and y coordinates specify the position of the top left corner of the window, in the coordinate system of the screen.
- As a final note, a stage is not visible on the screen until we show it by calling `stage.show()`
