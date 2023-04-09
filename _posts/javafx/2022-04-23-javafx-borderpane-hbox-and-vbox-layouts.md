---
layout: post
title: "JavaFX BorderPane HBox and VBox"
date: 2022-04-23 20:45:31 +0530
categories: "javafx"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/javafx/hbox-and-vbox/"
---

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

## BorderPane

BorderPane can be useful for displaying one large central component with four smaller components at left, top, right and bottoms

<img src="/assets/javafx/borderpane_hbox_and_vbox/borderpane.png" alt="borderpane">

> Note you can also design a border pane without setting the center, top or right vs... components, you are not restricted to put five components at all

BorderPane has two constructors:

```java

public BorderPane() {
  super();
}

/**
 * Creates an BorderPane layout with the given Node as the center of the BorderPane.
 * @param center The node to set as the center of the BorderPane.
 * @since JavaFX 8.0
 */
public BorderPane(Node center) {
  super();
  setCenter(center);
}
```

If you want to set child nodes as well:

```java
pane.setCenter(node);
pane.setTop(node);
pane.setRight(node);
pane.setBottom(node);
pane.setLeft(node);
```

A BorderPane sets the sizes of its child nodes:

- It can not set the height and width higher than the node's height and width. **This is the restriction**
- The top and bottom components will be displayed at their preffered heights, but their width is set equal to the full width of the container.
- The left and right components will be displayed at their preferred widths, but their height is set to the height of the
  container, minus the space occupied by the top and bottom components.
- The center component takes up any remaining space.

The default preffered size of BorderPane itself will be set to accommodate the preferred sizes of its children. (The minimum size will be calculated in a similary way, the maximum size is unlimited)

### What happens if child node has some restriction(s)?

In somehow, some components can not be fit into the space in the borderPane. For instance what would happen if child node, let's assume it's the center node, has no resize feature? (then, JavaFX couldn't fit the node to the available space). In that case(s):

- The center child node will be placed center of the available space
- The bottom child node will be placed at the bottom left corner of the bottom space in the pane.

We have a chance to change this behavior by calling static method:

```java
// position could be Pos.CENTER, POS.TOP LEFT, Pos.BOTTOM RIGHT etc..
BorderPane.setAlignment( child, position );
```

### Margin in BorderPane

We can also set the margin for the child nodes.

> A margin is empty space around the child

A margin is specified as a value of type Insets => `new Insets(top,right,bottom,left)`

We can apply margin to all child nodes while construction the borderPane, or we can set the margin to the individual child node:

```java
BorderPane.setMargin( child, insets );
```

## HBox and VBox

- HBox and Vbox are used to define components in a horizontal row or vertical row. They are both subclass of Pane
- We can use HBox layout in the bottom or top place of the BorderPane and we can use VBox layout in the left and right place of the BorderPane.

Now i will talk about HBox but same concept can be also applied VBox as well.

- We can add children in the HBox by calling `hbox.getChildren.add(node)` or `hbox.getChildren.addAll()`
- We can add space for each child in the HBox with `hbox.setSpacing( double amount )`
- By default, an HBox will resize its children to their preferred widths, and leaving some blank extra space on the right. (There is blank space because the width of the HBox will be larger than its preferred width)
  - If children's width is larger than the HBox's width, then HBox will shrink the children
- The height of the children node will be set to the full height in the HBox. We can stop the streching child nodes to fit the height of the HBox by calling: `hbox.setFillHeight(false).`. In that case HBox will use the preferred heights of its children.
- If we want to grow child nodes as HBox's width increases (in other words we want the children to grow beyond their preferred widths, to fill the available space in an HBox). To do that we need to set following layout constraint (it is a static method): `HBox.setHgrow( child, priority );`

  - `Priority.ALWAYS`, means that the child will always get a share of the extra space. Even we do this, The child’s width will still be limited by its maximum width, so we might need to increase the maximum width to get the child to grow to the extent that we want.

### To fill available space in HBox/VBox

Let's say we have buttons and we want the buttons to take all available space in HBox. First we need to set HGrow for each button. Second we have to set maximum width to double infinity for each button as well:

```java
HBox.setHgrow(button, Priority.ALWAYS);
HBox.setHgrow(anotherButton, Priority.ALWAYS);
button.setMaxWidth(Double.POSITIVE INFINITY);
anotherButton.setMaxWidth(Double.POSITIVE INFINITY);
```

If two button's preferred width are equal, extra space will be distributed equally:

```java
button.setPrefWidth(1000);
anotherButton.setPrefWidth(1000);
```

## Conclusion

- BorderPane can be useful for displaying one large central component with four smaller components at left, top, right and bottoms. There are also some restrictions while putting the child components.
- HBox and Vbox are used to define components in a horizontal row or vertical row. They are both subclass of Pane
  - To fill available space in HBox we need to set HGrow for each button. Second we have to set maximum width to double infinity for each button as well

I will continue with GridPane and TilePane ...
