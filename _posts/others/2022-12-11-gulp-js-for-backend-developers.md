---
layout: post
title: "gulpjs for backend developers"
date: 2022-12-11 12:45:31 +0530
categories: "others"
author: "mehmetozanguven"
---

This mini tutorial is especially for backend developers who implement also frontend part of the application inside the one package. Today, gulp seems outdated. However it still can be used in **many mini saas project** instead of learning new frontend framework. For instance you may use gulp with your Spring MVC project to increase development time.

> Even gulp increases the development time, learning curve, I think, is not easy.

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

<img src="/assets/others/gulpjs/gulp-js.png" alt="gulp-js" />

## What is Gulp ?

Gulp is a toolkit that can automize the Javascript workflow. For instance, gulp can detect and re-bundle when any javascript files has changed.

## How to add Gulp

First, you have to install node and npm into your computer. After that:

- Install the gulp-cli in global: `npm install --global gulp-cli`
- Create a project folder. For instance: **test**
- Initialize npm project with `npm init`
- Save the gulp: `npm install --save-dev gulp`

Finally your package.json should be like this one:

```json
{
  "name": "sample_gulp_project",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "gulp": "^4.0.2"
  }
}
```

After that step, create **gulpfile.js** file in the root directory (directoy that contains package.json)

### Example task

Update the gulpfile.js file:

```javascript
function defaultTask(cb) {
  // place code for your default task here
  cb();
}

exports.default = defaultTask;
```

Then run the `$ gulp` command from terminal:

```bash
$ gulp
[19:02:35] Using gulpfile ~/Desktop/temp/sample_gulp_project/gulpfile.js
[19:02:35] Starting 'default'...
[19:02:35] Finished 'default' after 2.04 ms
```

## Create a task

Gulp consists of task(s). And each task is an asynchronous JavaScript function.

Each task can be **public** or **private**

- **public** tasks can be run with `$ gulp `command
- **private** tasks can be used with `series()` and `parallel()`

For instance, in the following code snipped the task `clean()` is private because it isn't exported:

```javascript
const { series } = require("gulp");

// The `clean` function is not exported so it can be considered a private task.
// It can still be used within the `series()` composition.
function clean(cb) {
  // body omitted
  cb();
}

// The `build` function is exported so it is public and can be run with the `gulp` command.
// It can also be used within the `series()` composition.
function build(cb) {
  // body omitted
  cb();
}

exports.build = build;
exports.default = series(clean, build);
```

### Listing all tasks

If we want to see all tasks:

```bash
$ gulp --tasks
[19:14:51] Tasks for ~/Desktop/temp/sample_gulp_project/gulpfile.js
[19:14:51] ├── build
[19:14:51] └─┬ default
[19:14:51]   └─┬ <series>
[19:14:51]     ├── clean
[19:14:51]     └── build
```

### Creating task with gulp.task()

Instead of defining function we may use `gulp.task({taskname}, function)`

```javascript
const gulp = require("gulp");
gulp.task("myTask", () => {});
```

#### Defining the default task

Default task will be run for each gulp command:

```javascript
const gulp = require("gulp");
gulp.task("default", () => {});
```

And we can run our **internal tasks** in the defaul task. To do that we can pass the [tasks] instead of function:

```javascript
const gulp = require("gulp");
gulp.task("default", ["task1", "task2"]);

// tasks with default function
gulp.task("default", ["task1", "task2"], () => {});
```

### Combining tasks (with series and parallel)

If we want to run tasks:

- respectively, then we should use `series()`
- in parallel way, then we should use `parallel()`

```javascript
const { series, parallel } = require("gulp");

function transpile(cb) {
  // body omitted
  cb();
}

function bundle(cb) {
  // body omitted
  cb();
}

exports.build = series(transpile, bundle);
exports.anotherBuild = parallel(transpile, bundle);
```

```bash
$ gulp --tasks
[19:19:43] Tasks for ~/Desktop/temp/sample_gulp_project/gulpfile.js
[19:19:43] ├─┬ build
[19:19:43] │ └─┬ <series>
[19:19:43] │   ├── transpile
[19:19:43] │   └── bundle
[19:19:43] └─┬ anotherBuild
[19:19:43]   └─┬ <parallel>
[19:19:43]     ├── transpile
[19:19:43]     └── bundle
```

### Ending the task

We can end the task in different ways. Or we can pass the task's result to another task using `pipe()`

If we use `series()`, then if one of the task fails, then all remaining tasks will be canceled

If we use `parallel()`, then if one of the task fails, then all remaning tasks may run or fail.

### Using .pipe() and returning a stream

The .pipe() method will allow us to pipe together smaller single-purpose plugins or applications into a pipechain. This is what gives us full control of the order in which we would need to process our files.

```javascript
const { src, dest } = require("gulp");

function streamTask() {
  return src("*.js").pipe(dest("output"));
}

exports.default = streamTask;
```

## Working with folders and files

We should use `src()` and `desc()` to work with files/folders in the computer. These functions are provided by Gulp

#### src()

It transfer the given file to the stream which NodeJs can read. We may use pattern to match with appropriate files and all files will be transformed to the stream:

```javascript
const { src, dest } = require("gulp");

exports.default = function () {
  return src("src/*.js").pipe(dest("output/"));
};
```

##### Pattern

Here are the example patterns:

- `*.js`: Tries to find javascript files in the current directory. But it doesn't detect javascript files from folders such as script/index.js or script/nested/index.js
- `scripts/**/*.js`: Tries to find any javascript files in the current directory and all sub-directories Example: `scripts/index.js, scripts/nested/index.js,  scripts/nested/twice/index.js`

#### pipe()

We can use pipe(), if we wan to forward one task's result to another one.

#### dest()

The dest() method is used to set the output destination of our processed file.

## Gulp plugins

In essential, Gulp plugins are the Node Transform Streams. For instance gulp-uglify:

```javascript
const { src, dest } = require("gulp");
const uglify = require("gulp-uglify");
const rename = require("gulp-rename");

exports.default = function () {
  return (
    src("src/*.js")
      // The gulp-uglify plugin won't update the filename
      .pipe(uglify())
      // So use gulp-rename to change the extension
      .pipe(rename({ extname: ".min.js" }))
      .pipe(dest("output/"))
  );
};
```

## Gulp Watch

`watch()` as the name suggest watches given files. If anything changes, then it runs given tasks.

For instance the following code snippet will run `clean` and `javascript` tasks if anything changes in the src folder:

```javascript
const { watch, series } = require("gulp");

function clean(cb) {
  // body omitted
  cb();
}

function javascript(cb) {
  // body omitted
  cb();
}

function css(cb) {
  // body omitted
  cb();
}

exports.default = function () {
  watch("src/*.js", series(clean, javascript));
};
```

### gulp.task('watch')

If we want to run the watch command continuously, we can create 'watch' task and inside the task's function we can call `gulp.watch()`:

```javascript
const gulp = require("gulp");
const styleWatch = "./src/scss/**/*.scss";
const jsWatch = "./src/js/**/*.js";
gulp.task("default", gulp.series("scss-to-css", "js"));

gulp.task("watch", () => {
  gulp.watch(styleWatch, gulp.series("scss-to-css"));
  gulp.watch(jsSource, gulp.series("js"));
});
```

- `gulp.watch(styleWatch, ['scss-to-css'])` => watch() watches the files inside the `styleWatch`. If anything changes then it runs the `scss-to-css` task.
- `gulp.watch(jsSource, ['js'])` => watch() watches the files inside the `jsSource`. If anything changes then it runs the `js` task.

### gulp.watch and problem with reading the files

If we change the source files to `const styleWatch = "./src/scss/**/*.scss";`, then gulp can't watch the files and we have to restart again. Reason for that is using the global case `./` directory. Don't use `./` instead starts with `src/...`

## Examples

### Creating min.css file

In this example we are going to process style.css file and convert it to style.min.css file.

Because we are going to change file's name we need gulp-rename package: `npm install --save-dev gulp-rename`

```javascript
const gulp = require("gulp");
const rename = require("gulp-rename");
var styleSource = "./src/css/style.css";
var styleDist = "./dist/css/";
gulp.task("css-to-mincss", () => {
  return gulp
    .src(styleSource)
    .pipe(rename({ suffix: ".min" }))
    .pipe(gulp.dest(styleDist));
});
```

- `gulp.src(...)` => give the files that gulp searches
- `pipe()` => transfer src's output to another task
- `rename({ suffix: ".min" })` => separate the extension from the file and add **.min** next to it.
- `pipe()` => write the result after **rename** with **pipe** with **dest** operation dist/css

### Compile and minify SASS/SCSS

Our scss files:

- **src/scss/base/variables.scss**

```scss
$bg: #ccc;
```

- src/scss/styles.scss:

```scss
@import "base/variables";
body {
  background-color: $bg;
}
```

To convert scss-to-css we need a plugin 'gulp-sass' [https://www.npmjs.com/package/gulp-sass](https://www.npmjs.com/package/gulp-sass) => `npm install --save-dev gulp-sass sass`

```javascript
const gulp = require("gulp");
const rename = require("gulp-rename");
const sass = require("gulp-sass")(require("sass"));

var styleSource = "./src/scss/style.scss";
var styleDist = "./dist/css/";
gulp.task("scss-to-css", () => {
  return gulp
    .src(styleSource)
    .pipe(sass({ outputStyle: "compressed" }))
    .on("error", console.error.bind(console))
    .pipe(rename({ suffix: ".min" }))
    .pipe(gulp.dest(styleDist));
});
```

### gulp-Autoprefixer ve Sourcemap

#### Autoprefixer

While we are writing our css codes, we need to add prefix for each browser:

```css
a {
  -webkit-transition: all 320ms ease;
  -ms-transition: all 320ms ease;
  transition: all 320ms ease;
}
```

autoprefixer can do this for us automatically.

Install gulp plugin for autoprefixer => `npm install --save-dev gulp-autoprefixer`

And also we can list browsers we want to support => [https://github.com/browserslis](https://github.com/browserslis). Add "browserslist" in the package.json:

```json
{
    ...
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "gulp": "^4.0.2",
    "gulp-autoprefixer": "^8.0.0",
    "gulp-rename": "^2.0.0",
    "gulp-sass": "^5.1.0",
    "sass": "^1.54.5"
  },
  "browserslist": ["last 3 version", "> 1%", "IE 11"]
}
```

```javascript
const gulp = require("gulp");
const rename = require("gulp-rename");
const sass = require("gulp-sass")(require("sass"));
const autoprefixer = require("gulp-autoprefixer");

var styleSource = "./src/scss/style.scss";
var styleDist = "./dist/css/";
gulp.task("scss-to-css", () => {
  return gulp
    .src(styleSource)
    .pipe(sass({ outputStyle: "compressed" }))
    .on("error", console.error.bind(console))
    .pipe(autoprefixer())
    .pipe(rename({ suffix: ".min" }))
    .pipe(gulp.dest(styleDist));
});
```

#### Sourcemap

gulp plugin: `npm install --save-dev gulp-sourcemaps`

```javascript
const gulp = require("gulp");
const rename = require("gulp-rename");
const sass = require("gulp-sass")(require("sass"));
const autoprefixer = require("gulp-autoprefixer");
var sourcemaps = require("gulp-sourcemaps");

var styleSource = "./src/scss/style.scss";
var styleDist = "./dist/css/";
gulp.task("scss-to-css", () => {
  return gulp
    .src(styleSource)
    .pipe(sourcemaps.init())
    .pipe(sass())
    .on("error", console.error.bind(console))
    .pipe(autoprefixer())
    .pipe(rename({ suffix: ".min" }))
    .pipe(sourcemaps.write("./"))
    .pipe(gulp.dest(styleDist));
});
```

### Compile and bundle javascript es6 with babel

We have to install the following plugins/dependencies:

- **Browserify** => Enable to run NodeJS module onto browser. `npm install --save-dev browserify`
- **Babelify** => Browserify transform for Babel. `npm install --save-dev babelify @babel/core @babel/preset-env`

We should also add `preset` for the babel in the package.json file. `present` includes config for babel transforms.

Popular presets: **ES2015, Env, React** . For instance if we use ES2015, all javascript codes are transformed to es5.

```json
{
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "@babel/core": "^7.18.13",
    "@babel/preset-env": "^7.18.10",

    "sass": "^1.54.5"
  },
  "babel": {
    "presets": ["@babel/env"]
  },
  "browserslist": ["last 3 version", "> 1%", "IE 11"]
}
```

- **bundle** => Provided by Gulp to collect all js in one place
- **Source** => To convert back all files to stream (after the bundle operation), we need to install source dependency `npm install --save-dev vinyl-source-stream`. With this dependency we can continue to apply gulp operation. After this operation file's format would be vinly
- **Rename** => `npm install --save-dev gulp-rename`
- **Buffer** => Convert streaming vinyl files to use buffers. `npm install --save-dev vinyl-buffer`
- **Uglify**: `npm install --save-dev gulp-uglify`
- dist

```javascript
const gulp = require("gulp");
const rename = require("gulp-rename");
const sass = require("gulp-sass")(require("sass"));
const autoprefixer = require("gulp-autoprefixer");
const sourcemaps = require("gulp-sourcemaps");
const browserify = require("browserify");
const babelify = require("babelify");
const source = require("vinyl-source-stream");
const buffer = require("vinyl-buffer");
const uglify = require("gulp-uglify");

const jsFolder = "src/js/";
const jsSource = "script.js";
const jsDest = "./dist/js/";
gulp.task("js", () => {
  var b = browserify({ entries: jsFolder + jsSource, debug: true });
  return b
    .transform(babelify, { presets: ["@babel/env"] })
    .bundle()
    .pipe(source(jsSource))
    .pipe(rename({ extname: ".min.js" }))
    .pipe(buffer())
    .pipe(sourcemaps.init({ loadMaps: true }))
    .pipe(uglify())
    .pipe(sourcemaps.write("./"))
    .pipe(gulp.dest(jsDest));
});
```

In the next tutorial we are going to learn how to implement Spring MVC project with gulp.
