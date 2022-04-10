---
layout: post
title: "JavaFX Getting Started"
date: 2022-04-10 12:45:31 +0530
categories: "javafx"
author: "mehmetozanguven"
---

GUI programs are different from the traditional program that you have encountered. GUI programs are event-driven which means that these programs respond to the user actions such as clicking to the button, pressing a key on the keyboard etc ..

In Java, GUI programming is also(as expected) object-oriented. In other words, everything in the GUI program is object. For instance events are object, colors and fonts are objects.

In Java, there are several tools for gui programming, in these tutorial I will use JavaFx

> Note that JavaFx is no longer being distributed with (or as part of) Java Development Kit(JDK). Easiest way to program with JavaFx is to use Maven.

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

## JavaFx Application

A JavaFX program (or application) is represented by an object of type **Application**.

`Application` is an abstract class and includes one abstract method called `start()` .

To create a JavaFX program, you should extend `Application` class and provide a definition for the `start()` method.

In general, you should have main method to start javaFx application like this one:

```java
public static void main(String[] args) {
    launch(args);
}
```

When `main()` method is being executed, `launch()` method creates a new thread called **JavaFx Application Thread**. Anything that effects the GUI is done by the **JavaFx Application Thread**

`launch()` method creates an object and after that `start()` method is called on the JavaFx Application thread.

> `start()` method has the responsibility of the setup the GUI and open a window on the screen

## Create a sample JavaFx project with Maven

> When you are creating project via Maven, you do not need to download JavaFx SDK, you only need to specify to module in the pom.xml, Maven will download the necessary modules for the project.

JavaFx team has created maven archetypes to quickly create Maven project. Here are the examples for both Java 11 & 17 :

- For Java 11 (**but if your JDK version is 17, then you may encounter an exception, make sure that JDK version is also 11**):

```bash
mvn archetype:generate \
        -DarchetypeGroupId=org.openjfx \
        -DarchetypeArtifactId=javafx-archetype-simple \
        -DarchetypeVersion=0.0.6 \
        -DgroupId=org.openjfx \
        -DartifactId=sampleJavaFxProject \
        -Dversion=1.0.0 \
        -Djavafx-version=11
```

- For Java 17:

```bash
mvn archetype:generate \
        -DarchetypeGroupId=org.openjfx \
        -DarchetypeArtifactId=javafx-archetype-simple \
        -DarchetypeVersion=0.0.3 \
        -DgroupId=org.openjfx \
        -DartifactId=sampleJavaFxProject \
        -Dversion=1.0.0 \
        -Djavafx-version=17.0.1
```

- If you get an exception from the initial step, please refer to the javafx page [https://openjfx.io/openjfx-docs/#maven](https://openjfx.io/openjfx-docs/#maven)

After that you can open the project with your IDE (You can clear the default setup.)

Update the `App` class:

```java
import javafx.application.Application;
import javafx.application.Platform;
import javafx.geometry.Pos;
import javafx.scene.Scene;
import javafx.scene.control.Button;
import javafx.scene.control.Label;
import javafx.scene.layout.BorderPane;
import javafx.scene.layout.HBox;
import javafx.scene.text.Font;
import javafx.stage.Stage;

public class App extends Application {

    @Override
    public void start(Stage stage) {
        Label message = new Label("First JavaFx application");
        message.setFont(new Font(40));

        Button helloButton = new Button("Say hello");
        helloButton.setOnAction(e -> message.setText("Hello JavaFx"));

        Button goodByeButton = new Button("Say GoodBye");
        goodByeButton.setOnAction(e -> message.setText("GoodBye !!"));

        Button quitButton = new Button("Quit");
        quitButton.setOnAction(e -> Platform.exit());

        HBox buttonBar = new HBox(20, helloButton, goodByeButton, quitButton);
        buttonBar.setAlignment(Pos.CENTER);

        BorderPane root = new BorderPane();
        root.setCenter(message);
        root.setBottom(buttonBar);

        Scene scene = new Scene(root, 450, 200);
        stage.setScene(scene);
        stage.setTitle("JavaFX Application");
        stage.show();
    }

    public static void main(String[] args) {
        launch();
    }
}
```

<img src="/assets/javafx/getting_started/first_javafx.png" alt="first_javafx.png" />

## Stage, Scene and SceneGraph

### Stage

- A `Stage` object represents a window on the computer's screen.

The stage that is passed as a parameter to the `start()` method is constructed by the system. It represents the main window of a program. It is called **primary stage**.

- Even if stage(window) has been created by a system, it is empty and not visible. To make it visible you should call :`stage.show()`

The rest is just adding content to the window like this line:

```java
stage.setTitle("JavaFX Application");
```

### Scene

- A stage shows a **scene** which servers as a container for the GUI components in the window.

```java
stage.setScene(scene);
```

says that the scene will be displayed in the content area of the stage.

- A scene can be filled with GUI **components**, such as buttons and menu bars. Some components can be **container** for example `HBox`.
- A container represents a region in the window that can contain other components or containers.
- **A scene contains a single root component which is a container that contains all of the other components.**

### SceneGraph

- At the end all of these objects make up called the **scene graph** for the window.
- Here is the example for scene graph:

<img src="/assets/javafx/getting_started/scene_graph_example.png" alt="scene_graph_example.png" />

## Nodes and Layout

### Nodes

- Objects that can be part of scene graph are referred to as nodes. These objects have to belong to one of the subclasses of `javafx.scene.Node`
- Scene graph object that can act as containers must belong to one of the subclasses of `javafx.scene.Parent` (which of course `Parent` is a subclass of `Node` )

In our example Button object is a subclass of `Parent` (that's means button can actually contain other nodes)

### Layout

Containers are **Nodes** which can have other nodes as children.

The act of arranging a container's children on the screen is referred to as **layout**. In other words layout means setting the size and location of the components inside the container.

#### HBox

Different containers implement different layout policies. For example, an `HBox` is a container that simply arranges the components that it contains in a horizontal row.

```java
// First argument specifies the gap between the nodes
HBox buttonBar = new HBox( 20, helloButton, goodByeButton, quitButton);
// following line ceners the button within the HBox. Without it, they would be shoved over to the left edge of the window. Pos, short for position
buttonBar.setAlignment(Pos.CENTER);
```

#### BorderPane

A **BorderPane** is a container which implements different layout policy. It can contain up to five components, one in the center of the pane, one at the top, one at the bottom, one to the left and one to the right of the center.

## Events and Event Handlers

In our example, event occurs when the user clicks one of the buttons. Handling an event involves two objects:

- The event itself which holds the information about the event.
- `EventHandler` , a functional interface that defines a method `handle(e)` , where the parameter `e` is the event object.

To handle(or response) to an event, you should create a class and implement the EventHandler interface or directly use lambda expression.

For a button, `ActionEvent` is fired.

```java
// Generic EventHandler
@FunctionalInterface
public interface EventHandler<T extends Event> extends EventListener {
    void handle(T var1);
}

// ...

// Button event
public class App extends Application {
    @Override
    public void start(Stage stage) {
        Button helloButton = new Button("Say hello");
        helloButton.setOnAction(e -> message.setText("Hello JavaFx"));
    }
    // ...
}

public abstract class ButtonBase extends Labeled {
    public final void setOnAction(EventHandler<ActionEvent> var1) {
        this.onActionProperty().set(var1);
    }
}
```

## Platform.exit()

The static `exit()` method in the platform class is the preferred way to programmatically end a JavaFx program. It is preferred to `System.exit()` because it cleanly shuts down the application thread.

## Conclusion

- GUI programs are event-driven which means that these programs respond to the user actions such as clicking to the button
- GUI programming in Java is also object-oriented. Everything is object. For example, colors, fonts and events are objects
- JavaFX allows you to create Java applications with a modern, hardware-accelerated user interface that is highly portable.
- You may use JavaFX SDK itself, or you may use build tools (such as Maven, Gradle) to run JavaFX applications
- JavaFX program is represented by an object of type **Application** which is abstract class and includes one method called `start()`. To create a JavaFX program our main class have to extend `Application` class as well. And we have to provide definition for the `start()` method.
- Convenient way to call `start()` method is to call static void method called `launch()` in the main class:

```java
public static void main(String[] args) {
    launch(args);
}
```

- After `launch()` method is called, JavaFX will use the thread called **application thread** to render the ui.
- A **Stage** object represents a window on the computer's screen.
- A stage shows a **scene** which servers as a container for the GUI components in the window.
- Objects that can be part of sceneGraph are referred to as **node**s.
- The act of arranging a container's children on the screen is referred to as **layout**.
- Different containers implement different layout policies. (For example, `HBox`, `BorderPane` etc ...)
- Event occurs when the user clicks one of the buttons. Handling an event involves two objects: - The event itself (information about the event) - A function interface called `EventHandler`

Last but not least wait for the next one...
