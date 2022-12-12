---
layout: post
title: "Modern Spring Boot v3 with Thymeleaf, Tailwind and AlpineJS"
date: 2022-12-12 17:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

In this tutorial, we will integrate Spring MVC with gulp and webpack.

As you know creating Spring MVC project with Thymelaef project is so easy. But adding tailwind, npm packages or any related JS stuff is not easy.

<img src="/assets/spring/mvc/modern-spring-mvc.png" alt="modern-spring-mvc" />

> Note: In the previous [https://mehmetozanguven.github.io/others/2022/12/11/gulp-js-for-backend-developers.html](https://mehmetozanguven.github.io/others/2022/12/11/gulp-js-for-backend-developers.html), I explained what the Gulp is.

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

Here is the list of technologies we will use:

<img src="/assets/spring/mvc/list-of-technologies.png" alt="list-of-technologies" />

## Github Link

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/spring-boot-examples/tree/master/thymeleaf-with-webpack)

<br />

Let's walk through each step one by one.

## Installing Node and NPM

Please install the Node and NPM, because we will create project with `npm commands`

## Initialize Spring project

Via [https://start.spring.io/](https://start.spring.io/), please create the new spring boot project with the following dependencies:

- Web, Thymeleaf

## Create Controller, Html Page, JS and CSS files

Please create the following files:

- **IndexController.java**

```java
@Controller
public class IndexController {

    @GetMapping("/")
    public String getIndex() {
        return "index";
    }
}
```

- **util.js** (resources/static/js/util/util.js)

```javascript
export const utilFunction = () => {
  console.log("Util function called");
};
```

- **index.js** (resources/static/js/index.js)

```javascript
import { utilFunction } from "./util/util";

utilFunction();
```

- **index.scss** (resources/static/css/index.scss)

```scss
$primary-color: #57f357;

#dummy {
  color: $primary-color;
}
```

- **index.html** (resources/templates/index.html)

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
  <head>
    <meta charset="UTF-8" />
    <title>Title</title>
    <link rel="stylesheet" th:href="@{/css/index.css}" />
  </head>
  <body>
    <div id="dummy">Test</div>
    <script th:src="@{/js/index.js}"></script>
  </body>
</html>
```

## Update application-local.properties file

> If you skip this step, you will probably be lost in the configuration !!

We have to disable thymeleaf cache in the local environment. **Otherwise your changes won't be reflected to the browser.** Update the application-local.properties :

```properties
spring.thymeleaf.cache=false
spring.web.resources.chain.cache=false
```

Note: When you run project from IDEA (such as IntelliJ), your environment will be **default**. I can't say how to change your environment in Eclipse or other IDEAs. However I can show the configuration for IntelliJ:

- Open **Run/Configurations** Panel
- Add VM options (click Modify Options -> Add VM options)
- Then write **-Dspring.profiles.active=local**

<img src="/assets/spring/mvc/modern-mvc-intellij-config.png" alt="modern-mvc-intellij-config" />

## Exclude directories for maven build

Because gulp & webpack will build the appropriate HTML, JS & CSS files, we have to tell Maven do not build these files. We can do that by updating our `pom.xml` file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project ...>
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>3.0.0</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	...
	<description>Demo project for Spring Boot</description>
	<properties>
		<java.version>17</java.version>
	</properties>
	<dependencies>
		...
	</dependencies>

	<build>
		<resources>
			<resource>
				<directory>src/main/resources</directory>
				<excludes>
					<exclude>**/*.html</exclude>
					<exclude>**/*.css</exclude>
					<exclude>**/*.scss</exclude>
					<exclude>**/*.js</exclude>
				</excludes>
			</resource>
		</resources>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				<configuration>
					<excludes>
						<exclude>
							<groupId>org.projectlombok</groupId>
							<artifactId>lombok</artifactId>
						</exclude>
					</excludes>
				</configuration>
			</plugin>
		</plugins>
	</build>
</project>
```

With this configuration, if you run `mvn clean install -DskipTests`, then target directory will not contain any html, js or css files

> Do not forget the reload your maven configuration !!

## Add maven-frontend-plugin

This plugin [https://github.com/eirslett/frontend-maven-plugin](https://github.com/eirslett/frontend-maven-plugin) will allow us to download Node and NPM locally for our project and we can be able run the npm command while building our application with maven

Update the `pom.xml` (with the appropriate nodejs and npm version):

> Do not change `gulp build` command, because we are going to define this command in our `package.json` file

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project ...>
	...
	<properties>
		<java.version>17</java.version>
		<frontend.maven.plugin.version>1.12.1</frontend.maven.plugin.version>
		<node.version>v16.16.0</node.version>
		<npm.version>8.11.0</npm.version>
	</properties>
	<dependencies>
		...
	</dependencies>

	<build>

		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				...
			</plugin>

			<plugin>
				<groupId>com.github.eirslett</groupId>
				<artifactId>frontend-maven-plugin</artifactId>
				<version>${frontend.maven.plugin.version}</version>

				<executions>
					<execution>
						<id>install node and npm</id>
						<goals>
							<goal>install-node-and-npm</goal>
						</goals>
						<configuration>
							<nodeVersion>${node.version}</nodeVersion>
							<npmVersion>${npm.version}</npmVersion>
						</configuration>
					</execution>
					<execution>
						<id>npm install</id>
						<goals>
							<goal>npm</goal>
						</goals>
						<configuration>
							<arguments>install</arguments>
						</configuration>
					</execution>
					<execution>
						<id>run-gulp-build</id>
						<goals>
							<goal>gulp</goal>
						</goals>
						<configuration>
							<arguments>build</arguments>
						</configuration>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
</project>
```

## Create release profile for production build

When we are ready to release our application, we should create a new jar with the production flag: `mvn clean install -Prelease -DskipTests`

Please update `pom.xml`:

> Do not change `gulp build --env production` command !!

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project ...>
	<properties>
        ...
	</properties>
	<dependencies>
		...
	</dependencies>

	<build>
		<resources>
            ...
		</resources>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				....
			</plugin>

			<plugin>
				<groupId>com.github.eirslett</groupId>
				<artifactId>frontend-maven-plugin</artifactId>
				<version>${frontend.maven.plugin.version}</version>
                ...
			</plugin>
		</plugins>
	</build>
	<profiles>
		<profile>
			<id>release</id>
			<build>
				<resources>
					<resource>
						<directory>src/main/resources</directory>
						<excludes>
							<exclude>**/*.html</exclude>
							<exclude>**/*.css</exclude>
							<exclude>**/*.scss</exclude>
							<exclude>**/*.js</exclude>
						</excludes>
					</resource>
				</resources>
				<plugins>
					<plugin>
						<groupId>com.github.eirslett</groupId>
						<artifactId>frontend-maven-plugin</artifactId>
						<executions>
							<execution>
								<id>run-gulp-build</id>
								<goals>
									<goal>gulp</goal>
								</goals>
								<configuration>
									<arguments>build --env production</arguments>
								</configuration>
							</execution>
						</executions>
					</plugin>
				</plugins>
				<finalName>your_final_jar_name</finalName>
			</build>
		</profile>
	</profiles>
</project>
```

> **Make sure that you have reloaded your maven project. Otherwise your pom.xml setup may not work.**

## Initialize npm project

> Before create new project make sure that you have installed gulp-cli globally.

In the Spring Boot Project folder, run the following command: `npm init`

After that you should see new file called `package.json`:

```json
{
  "name": "demo",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC"
}
```

## Create gulpfile.js and postcss.config.js files

Make sure that these files are placed in the same directory with `package.json`

- gulpfile.js => Include gulp's tasks (empty file)
- postcss.config.js => Include configuration for postcss (empty file)

At the end, you should have these files:

- projectFolder/package.json
- projectFolder/gulpfile.js
- projectFolder/postcss.config.js

## Install npm dependencies

I am not going to explain reason for each dependencies. You can search on the Internet:

```bash
npm install --save-dev webpack webpack-stream webpack-cli babel-loader @babel/core @babel/cli @babel/preset-env vinyl-named autoprefixer gulp gulp-postcss gulp-sass node-sass gulp-environments gulp-html-minifier-terser browser-sync
```

- webpack-stream : Integration gulp with webpack [https://github.com/shama/webpack-stream](https://github.com/shama/webpack-stream)
- gulp-sass & node-sass: Converting scss, sass files to css files. Detail explanation: [https://marioyepes.com/use-gulp-and-webpack-toghether/](https://marioyepes.com/use-gulp-and-webpack-toghether/)
- gulp-html-minifier-terser: To minify html
- gulp-environments: Using environment in gulpfile.js [https://www.npmjs.com/package/gulp-environments](https://www.npmjs.com/package/gulp-environments)
- browser-sync: Keep multiple browsers & devices in sync when building websites

## Update postcss.config.js

Add the following content:

```javascript
module.exports = {
  plugins: {
    autoprefixer: {},
  },
};
```

## Update gulpfile.js

Now we have to define tasks in the gulpfile.js

- First import the libraries:

```javascript
const gulp = require("gulp");
const named = require("vinyl-named");
const webpack = require("webpack-stream");
const environments = require("gulp-environments");
const postcss = require("gulp-postcss");
const sass = require("gulp-sass")(require("node-sass"));
const browserSync = require("browser-sync");
const htmlmin = require("gulp-html-minifier-terser");
```

- Create browserSync instance:

```javascript
// ...
const htmlmin = require("gulp-html-minifier-terser");

browserSync.create();
```

- Get the current environment as **development & production**. When we call **development()**, it will return true if the environment is development otherwise it will return false. (The same thing will be applied for the production)

```javascript
// ...
const environments = require("gulp-environments");
// ...
browserSync.create();

const development = environments.development;
const production = environments.production;
```

- Because spring mvc loads the html, css and js files from the directory `target/classes`, let's define a variable for that:

```javascript
// ...
const production = environments.production;

// Our gulp's task will update the folder targetClassesDestination
const targetClassesDestination = "target/classes/";
```

- Create html task

Copy the html files from the source (which is placed src/main/resource) and apply some operations (such as minify) then put the result to the folder:

```javascript
// ...
const targetClassesDestination = "target/classes/";

/// HTML tasks
const htmlSource = ["src/main/resources/**/*.html"];
function copyHtmlTask(done) {
  gulp
    .src(htmlSource)
    .pipe(htmlmin({ collapseWhitespace: true }))
    .pipe(gulp.dest(targetClassesDestination));
  done();
}
```

- Create SCSS and CSS task

```javascript
// ...

const targetClassesDestination = "target/classes/";

// ...

/// CSS & scss tasks
const scssSources = [
  "src/main/resources/**/*.css",
  "src/main/resources/**/*.scss",
];
function copyScssTask(done) {
  gulp
    .src(scssSources)
    .pipe(postcss())
    .pipe(sass())
    .pipe(gulp.dest(targetClassesDestination));
  done();
}
```

- Create Javascript task

This will be different than others. We have to specify more exact location for the `.src()` and `.dest()` stream(Because of the customized setup). Otherwise gulp would place output files to the different location.

```javascript
// ...

const targetClassesDestination = "target/classes/";

// ...

/// JS tasks
const jsSources = ["src/main/resources/static/js/**/*.js"];

const jsOutput = "target/classes/static/js";
function copyJsModern(done) {
  gulp
    .src(jsSources)
    .pipe(named())
    .pipe(
      webpack({
        devtool: development() ? "inline-source-map" : false,
        mode: development() ? "development" : "production",
        module: {
          rules: [
            {
              test: /\.js$/,
              exclude: /node_modules/,
              loader: "babel-loader",
              options: { presets: ["@babel/preset-env"] },
            },
          ],
        },
      })
    )
    .pipe(gulp.dest(jsOutput));
  done();
}
```

Now the rest is just to find an answer to a question "how to run these internal task"

- Create watch task for gulp:

We will define this task as default.

```javascript
// ...

/// JS tasks
const jsSources = ["src/main/resources/static/js/**/*.js"];

// ...

function watching_files() {
  browserSync.init({
    proxy: "localhost:8080", // default location for spring boot
    injectChanges: false,
    files: [
      "target/classes/templates",
      "target/classes/static/js",
      "target/classes/static/css",
    ],
  });
  gulp.watch(htmlSource, gulp.series(copyHtmlTask, copyScssTask));
  gulp.watch(scssSources, gulp.series(copyScssTask));
  gulp.watch(jsSources, gulp.series(copyJsModern));
}
gulp.task("watch", watching_files);
gulp.task("default", gulp.series("watch"));
```

- `gulp.watch(htmlSource, gulp.series(copyHtmlTask, copyScssTask));` => listen the files inside the `htmlSource`, if anything changes then run the `copyHtmlTask & copyScssTask` tasks respectively
- `gulp.watch(scssSources, gulp.series(copyScssTask));` => listen the files inside `scssSources`, if anything changes then run `copyScssTask`task
- `gulp.watch(jsSources, gulp.series(copyJsModern));` => listen the files inside `jsSources`, if anything changes then run `copyJsModern`task
- `gulp.task("watch", watching_files);` => sets watching_files function as watch command
- `gulp.task("default", gulp.series("watch"));` => if anyone runs the gulp command, by default watch command will be run.

- Create build task for gulp

After defining the watch task, we also have to define our build task flow:

```javascript
// ...
gulp.task("build", gulp.series(copyHtmlTask, copyScssTask, copyJsModern));
```

Nothing but just run the all tasks by one by.

## Define npm scripts for building the application

Update the script field in the `package.json`:

```json
{
  "name": "demo",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "gulp build",
    "watch": "NODE_ENV=development gulp watch",
    "build-prod": "NODE_ENV='production' gulp build"
  },
  "author": "",
  "license": "ISC"
  // ...
}
```

At this stage, you can be able to run your spring application without any error. Here are the steps to run application:

- First run the spring boot application from IDEA. (Don't forget to set profile to local, otherwise resources will be cached)
- Then run the following npm command: `npm run build && npm run watch`

After that your browser will open the **http://localhost:3002**.

> To check everything works as expected, while application is running, change anything in the index.html, index.scss or index.js file. You should see the changes immediately.

To build the application, just stop npm command and run: `mvn clean install -Prelease -DskipTests` and run the jar file: `java -jar target/jar_name.jar`

> If you have prod environment: `java -jar -Dspring.profiles.active=prod target/jar_name.jar`

Let's add tailwind and alpineJs

## Add TailwindCSS

For more information about tailwindcss => [https://tailwindcss.com/docs/installation](https://tailwindcss.com/docs/installation)

- Install the following packages:

```bash
npm install -D tailwindcss
npx tailwindcss init
```

- Create the **tailwind.config.js** file in the projectFolder

```javascript
module.exports = {
  content: ["./src/main/resources/templates/**/*.html"],
  theme: {
    extend: {},
  },
  plugins: [],
};
```

- Create `style.tailwind.css` file inside the resources/static/css

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

- Update postcss.config.js file:

```javascript
module.exports = {
  plugins: {
    tailwindcss: {}, // add tailwindcss
    autoprefixer: {},
  },
};
```

- Update the index.html (to verify everything is okey):

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
  <head>
    ...
    <link rel="stylesheet" th:href="@{/css/style.tailwind.css}" />
  </head>
  <body>
    <div class="text-4xl" id="dummy">Test</div>
    <script th:src="@{/js/test.js}"></script>
  </body>
</html>
```

Re-run the project.

## Add AlpineJS

- Install the npm package: `npm install alpinejs`

- Update the **index.html**:

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
  <head>
    ...
    <style>
      [x-cloak] {
        display: none;
      }
    </style>
  </head>
  <body>
    <div x-data="dummyAlpineComponent" class="text-4xl" id="dummy">Test</div>
    ...
    <script th:src="@{/js/dummyComponent.js}"></script>
  </body>
</html>
```

- Create javascript file for alpineComponent (resources/static/js/dummyComponent.js)

```javascript
import Alpine from "alpinejs";

Alpine.data("dummyAlpineComponent", () => ({
  init() {
    console.log("Alpine works !! ");
  },
}));

window.Alpine = Alpine;

Alpine.start();
```

Re-run the project and open the home page, you can see the log statement "Alpine works !!"

As a final note, because you now have node_modules directory, IntelliJ will try to create an index anything in the node_modules. If you want to disable this:

- Open Project Structure -> Modules
- Mark node and node_modules as **excluded folders**

<img src="/assets/spring/mvc/intellij_excluded_folders.png" alt="intellij_excluded_folders" />

You may say that "too much steps to just configure and run the project." You are definitely right. And this is also time consuming. I hope in the future release of Spring Boot, Spring's team provides easy and elegant way.

But until then, you have to do it yourself !!!
