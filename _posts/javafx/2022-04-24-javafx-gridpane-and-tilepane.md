---
layout: post
title: "JavaFX GridPane and TilePane"
date: 2022-04-24 20:45:31 +0530
categories: "javafx"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/javafx/gridpane-and-tilepane/"
---

## GridPane

<img src="/assets/javafx/gridpane-and-tilepane/gridpane.png" alt="gridpane" />

- Layout children nodes in rows and columns
- Rows and columns are numbered, starting from zero.
- Each rows and columns have not the same width and height
- We can also add gaps between each row and columns:

```java
grid.setHGap(size);
gris.setVGap(size);
```

- We can specify the row and column while adding the child node: `grid.add(child, column, row);`
  - If child node will occupy more than one column and row we can do this by: `grid.add(child, column, row, columnCount, rowCount);`
- A GridPane will resize each child to fill the position or positions that it occupies in the grid (within minimum and maximum size limits)
- The preferred width of a column will be just large
  enough to accommodate the preferred widths of all the children in that column, and similarly for the preferred height.
- We can also add some restriction for column width or row height. By default it will be preferred size of the child component in that (row,column). For instance if we want to have specific height for each row(we can also set height as percentage):

```java
gridpane.getRowConstraints().addAll(
new RowConstraints(100), // row 0 has height 100 pixels
new RowConstraints(150), // row 1 has height 150 pixels
new RowConstraints(100), // row 2 has height 100 pixels
new RowConstraints(200), // row 3 has height 200 pixeds
);
```

- When percentages are used, the grid pane will expand to fill available space, and the **row height** or **column width** will be computed from the percentages.

For example, to force a five-column gridpane to fill the available width and to force all columns to have the same size:

```java
for (int i = 0; i < 5; i++) {
    ColumnConstraints constraints = new ColumnConstraints();
    constraints.setPercentWidth(20);
    gridpane.getColumnConstraints().add(constraints);
}
```

## TilePane

If we want each (row, column) pair to take equal size, then we can use TilePane. We can set the number of columns and rows by calling: `tpane.setPrefColumns(cols); & tpane.setPrefRows(rows);`

For instance if we set column to 1 then we actually get a VBox, and if we set row to 1 then we actually get an HBox.

Some constructors in the TilePane:

```java
public TilePane() {
  super();
}
public TilePane(double hgap, double vgap) {
  super();
  setHgap(hgap);
  setVgap(vgap);
}

public TilePane(Node...children) {
  super();
  getChildren().addAll(children);
}
```
