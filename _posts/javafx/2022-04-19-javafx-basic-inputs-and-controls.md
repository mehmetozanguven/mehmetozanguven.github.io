---
layout: post
title: "JavaFX Basic Inputs and Controls"
date: 2022-04-19 20:45:31 +0530
categories: "javafx"
author: "mehmetozanguven"
---

If you are going to draw something on the screen and handle some mouse event, then you can only use MouseEvent listener and AnimationTimer that's it!! But in one way or another, most probably you will need to add some inputs for user's interactivity. For that you may want to use Checkbox, RadioButton, TextField and more ... Let's look at the some of these objects:

## Label and Button

These two components display text to the user. User can view but not edit.

Actually objects that is part of the user interface but not editable extends object called `Labeled`. For instance:

- Label class extends Labeled:

```java
public class Label extends Labeled {}
```

- And also button class extends Labeled:

```java
public class Button extends ButtonBase{}
public abstract class ButtonBase extends Labeled{}
```

`Labeled` class defines the following methods:

- `setGraphic(node)`: sets the control’s graphical element. For instance: we can use this method when we want to add image to the button
- `setText(string)`: sets the text that is displayed on the control. Text can be also multi-line using the new line character `"\n"`
- `setFont(font)`: sets the font for the text
- `setTextFill(color)`: sets the paint that is used for drawing the text.
- `setGraphicTextGap(size)`: sets the amount of space between the text and the graphic.
- `setContentDisplay(displayCode)`: set where the graphic should be places with respect to the next. Examples: `ContentDisplay.LEFT, ContentDisplay.RIGHT, ContentDisplay.TOP`

### Label

- Its purpose is to display text(which is not editable) to the user.
- It has two constructors:

```java
Label onlyText = new Label("Hello");
Label textWithIcon = new Label("Choose Icon", myIconObject);
```

- In default there is no padding, colors etc.. on the label. We can add it:

```java
message.onlyText("-fx-border-color: green; -fx-border-width: 12px; " +
"-fx-background-color: white; -fx-padding: 3px");
```

### Button

- Like a `Label`, it displays text to the user and user can click the button as well.
- Button has also two constructors:

```java
Button onlyText = new Button("Text");
Button buttonWithIcon = new Button("Another Button", myIconObject);
```

- When user clicks a button, `ActionEvent` is generated. We can register an event handler for the button's action using `setOnAction` method:

```java
buttonWithIcon.setOnAction( e -> System.out.println("...") );
```

- Button has also additional methods such `setDisable(boolean)` and `setToolTip(string)`
- If we want to trigger an event for the specific button, when user presses the **enter** key on the keyboard, we can do this also:

```java
button.setDefaultButton(true);
```

## CheckBox and RadioButton

### Checkbox

- It has two states: selected(checked) or unselected(unchecked).
- It is also a subclass of `Labeled`
- It has only one constructor:

```java
CheckBox test = new CheckBox("Check Test");
```

- We can get state of the checkbox: `test.isSelected()` (returns true is selected otherwise false)
- We can also set the state programmatically by calling `test.setSelected(boolean)`
- In general we don't need to register an event for the checkbox because `isSelected()` method will return the state. If we want to also listen the state changes in the checkbox we can register a listener with `test.setOnAction(e -> ...)`

> Note: `setSelected()` method doesn't trigger an action. If we want to simulate user event, then we can call `fire()` method.

### RadioButton

- Like a checkbox, a radio button can be either selected or not.
- If we are going to use RadioButton in isolation, it acts like a Checkbox. But in general we use RadioButton in a group (with other radioButtons)
- To group radio buttons, we use an object called `ToggleGroup`.
  - `ToggleGroup` is not component and can be visible as itself on the screen
  - It is used for grouping the RadioButtons
- After creating the radioButtons we can add these to the same toggleGroup. Here is the example:

```java
ToggleGroup radiouGroup = new ToggleGroup();

RadioButton redButton = new RadiouButton("Red");
redButton.setToggleGroup(radiouGroup);

RadioButton blueButton = new RadiouButton("Blue");
blueButton.setToggleGroup(radiouGroup);

RadioButton greenButton = new RadiouButton("Green");
greenButton.setToggleGroup(radiouGroup);

// initiality redButton is selected
radiouGroup.selectToggle(redButton);
```

If we want to learn selected radioButton:

```java
Toggle selection = radiouGroup.getSelectedToggle();
if (selection == redButton) {}
else if (selection == blueButton) {}
...
```

## TextField and TextArea

- These are editable components
- **TextField** holds single line
- **TextArea** holds multiple lines
- Both components have the following methods:
  - `setText(text)` & `getText()`
  - `appendText(text)`
  - `setFont(font)`
  - `setEditable(boolean)`: if we set to false then user can't modify the inputs
- **TextField** has a number of columns(in default 12) to determine in size. We can change it with `input.setPrefColumnCount(n)`
- **TextArea** has both number of columns and number of rows. We can change it with: `setPrefColumnCount(n)` & `setPrefRowCount(n)`
- **TextArea** has also method called `setWrapText(boolean)` (in default it is false) to wrap longer text to the next line

## Conclusion

- You can use Label and Button to display text to the user. User can view but not edit.
- Objects that is part of the user interface but not editable extends object called `Labeled`.
- **Label** purpose is to display text(which is not editable) to the user.
- Like a `Label`, **Button** displays text to the user and user can click the button as well.
- **Checkbox** has two states: selected(checked) or unselected(unchecked).
  - We can get state of the checkbox: `test.isSelected()`
- Like a checkbox, a radio button can be either selected or not.
  - To group radio buttons, we use an object called `ToggleGroup` which is not component and can be visible as itself on the screen. ToggleGroup is used for grouping the RadioButtons
- **TextField and TextArea** are editable components.
  - **TextField** holds single line
  - **TextArea** holds multiple lines

Last but not least wait for the next one ...
