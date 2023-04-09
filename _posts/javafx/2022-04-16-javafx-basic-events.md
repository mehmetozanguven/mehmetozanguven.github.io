---
layout: post
title: "JavaFX Basic Events"
date: 2022-04-16 12:45:31 +0530
categories: "javafx"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/javafx/basic-events/"
---

In JavaFX, events are represented by objects. When an event occurs, system collects all the information relevant to the event and construct an object to contain that information.

Different types of events are represented by different objects. For instance mouse events are represented by `MouseEvent` object.

After the event object is constructed, it can be passed as a parameter to a designated method. That method is called an **event handler**. In JavaFX, event handler are generally written as functional interface.

> Even if you don't care about, there are actually many processes to handle event. System handles these processing stuff. However you should (or must) know that there is a kind of a loop which does the following:
>
> ```wiki
> while the program is still running:
> 	Wait for the next event to occur
> 	Handle the event
> ```
>
> This loop is called an **event loop**. Every GUI program has an event loop. In Java, you don't have to write the loop. System will handle it.

## Event Handling

To act on the event, a program must detect the event. In order to detect an event, the program must **listen** for it. Listening can be done by registering an **event listener**

For many kinds of events in JavaFX, listeners are defined by the functional interface called `EventHandler` (which defines the method `handle(event)`)

> When you provide a definition for the `handle(event)` method, you write the code that will be executed to handle the event

Here is the example for registration for an event:

```java
helloButton.setOnAction(e -> message.setText("something"));
/*
When the user click the button, an event of type ActionEvent` is generated.
The target of that event is the helloButton
The method helloButton.setOnAction() registers an event listener which will receive notification of any ActionEvent from the button
*/
```

Also handler can consume an event which basically stops the any further handling of the event. For instance:

```java
@Override
public void start(Stage primaryStage) {
    Rectangle rect = new Rectangle(50, 50);

    StackPane root = new StackPane(rect);

    rect.addEventFilter(MouseEvent.MOUSE_CLICKED, evt -> {
        System.out.println("rect click(filter)");
//      evt.consume();
    });
    root.addEventFilter(MouseEvent.MOUSE_CLICKED, evt -> {
        System.out.println("root click(filter)");
//        evt.consume();
    });

    root.setOnMouseClicked(evt -> {
        System.out.println("root click(handler)");
//      evt.consume();
    });
    rect.setOnMouseClicked(evt -> {
        System.out.println("rect click(handler)");
//      evt.consume();
    });

    Scene scene = new Scene(root, 200, 200);

    primaryStage.setScene(scene);
    primaryStage.show();
}
```

When you click on `rect` , the event handling starts at the `root Node` .In this case filter will be called. If the event is not consumes in the filter, it is then passed to the `rect Node`. If that filter also doesn't consume the event, then it is passed to the event handler of the `rect` . If you want to stop this movement, you can just uncomment the evt.consume in any stage.

## Mouse Events

To register a mouse event is actually same as the registration of the button action. It would be good to know that there are many mouse events:

- Clicking a mouse generates three events: "mouse pressed" event, "mouse released" event and "mouse clicked" event.
- Simply moving the mouse generates a sequence of events, you can register moving event with `c.setOnMouseMoved(handler)`
- There are also many mouse event types:
  - `MouseEntered` , generated when the mouse cursor moves from outside a component into the component.
  - `MouseExited`, generated when the mouse cursor moves out of the component.
  - `MousePressed`, generated when one of button pressed on the mouse.
  - `MouseReleased` , generated when the user releases one of the buttons on the
    mouse
  - `MouseClicked`, generated after a mouse released event if the user pressed and released
    the mouse button on the same component
  - `MouseDragged`, generated when the user moves the mouse while holding down a mouse button
  - `MouseMoved`, generated when the user moves the mouse without holding down a button.

User can hold down certain modifier keys while using the mouse. The possible modifier keys could be: the shift key, the control key etc.. You can respond to an event according to the modifier key. Examples:

```java
evt.isShiftDown(), evt.isControlDown(), evt.isAltDown()
```

You can also want to return different responses according to the pressed button on the mouse. For instance you can listen only left button of the mouse, or middle etc.. If you want to know which button was pressed or released, you can call this method: `evt.getButton()` which returns one of the enumerated values: `MouseButton.PRIMARY, MouseButton.MIDDLE, or MouseButton.SECONDARY`

```java
// In this example, if the user pressed mouse on the canvas without pressing shiftkey, rectangle will be drawn with red color, otherwise rectangle will be blue color
canvas.setOnMousePressed(evt -> {
    GraphicsContext g = canvas.getGraphicsContext2D();
    if (evt.isShiftDown()) {
        g.setFill(Color.BLUE);
        g.fillOval(...)
    } else {
        g.setFill(Color.RED);
        g.fillOval(...)
    }
});
```

## AnimationTimer

This is kind of a basic event. This event is used to drive an animation. The event in this case happen in the background, and you don't have to register a listener to respond to them. Only you need to write a method that will be called by the system when the event occurs.

> A computer animation is just a sequence of images, presented to the user one after
> the other. If the time between images is short, and if the change from one image to another
> is not too great, then the user perceives continuous motion.

In JavaFX, you can program an animation using an object of type `AnimationTimer` . You can run the animation with `animation.start()` and stop it via `animation.stop()`

Animation has a method called `handle(time)` , but this is not the method that you call, it’s one that you need to write to say what happens in the animation. The system will call your `handle()` method.

The handle() method will be called on the JavaFX application thread. Since JavaFX animations are meant to run at 60 frames per second, which means handle() will ideally be called every 1/60 second.

## Conclusion

- In JavaFX, events are represented by objects. When an event occurs, system collects all the information relevant to the event and construct an object to contain that information.
  - After the event object is constructed, it can be passed as a parameter to a designated method. That method is called an **event handler**.
  - To act on the event, a program must detect the event. In order to detect an event, the program must **listen** for it. Listening can be done by registering an **event listener**
- To register a mouse event is actually same as the registration of the button action.
  - Clicking a mouse generates three events: "mouse pressed" event, "mouse released" event and "mouse clicked" event.
- AnimationTimer is kind of a basic event. This event is used to drive an animation. The event in this case happen in the background, and you don't have to register a listener to respond to them.
