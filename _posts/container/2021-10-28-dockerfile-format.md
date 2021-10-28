---
layout: post
title: Dockerfile Format and Commands
date: 2021-10-28 13:45:31 +0530
categories: "container"
author: "mehmetozanguven"
---

Before diving into the podman or docker itself, we should first know how to create dockerfile. Because writing a Dockerfile is the first step to containerize an application.

We can think of these Dockerfile commands as a step by step recipe on how to build up your image.  

> Dockerfiles describe how to assemble a private filesystem for a container, and can also contain some metadata describing how to run a container based on this image.

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

## Some Dockerfile commands

### FROM

- **Dockerfile must begin with a `FROM` instruction** 
- The `FROM` instruction specifies the **parent image** from which we are building.

### COPY

````dockerfile
COPY [--chown=<user>:<group>] <src>... <dest>
COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]
````

- The `COPY` instruction copies new files or directories from `<src>` and adds them to the filesystem of the container at the path `<dest>`.
- The `<dest>` is an absolute path, or a path relative to `WORKDIR`, into which the source will be copied inside the destination container.

```dockerfile
# uses a relative path, and adds “test.txt” to <WORKDIR>/relativeDir/:
COPY test.txt relativeDir/
# uses an absolute path, and adds “test.txt” to /absoluteDir/
COPY test.txt /absoluteDir/
```

### ARG

- Syntax:  `ARG <name>[=<default value>]`
- The `ARG` instruction defines a variable that users can pass at build-time to the builder with the `docker build` command using the `--build-arg <varname>=<value>` flag. 
- Examples:

```dockerfile
FROM busybox
ARG user1
ARG buildno
ARG user=someuser # with default value
ARG buildno=1 # with default value
```

```bash
$ docker build --build-arg user=myUser .
```

### EXPOSE

```dockerfile
EXPOSE <port> [<port>/<protocol>...]
```

- The `EXPOSE` instruction informs Docker that the container listens on the specified network ports at runtime.
- You can specify whether the port listens on TCP or UDP, and the default is TCP if the protocol is not specified.

### ENTRYPOINT

````dockerfile
# The exec form, which is the preferred form:
ENTRYPOINT ["executable", "param1", "param2"]
# The shell form:
ENTRYPOINT command param1 param2
````

- An `ENTRYPOINT` allows us to configure a container that will run as an executable.

### WORKDIR

```dockerfile
WORKDIR /path/to/workdir
```

- The `WORKDIR` instruction sets the working directory for any `RUN`, `CMD`, `ENTRYPOINT`, `COPY` and `ADD` instructions that follow it in the `Dockerfile`.
- If the `WORKDIR` doesn’t exist, it will be created even if it’s not used in any subsequent `Dockerfile` instruction.



## Difference Between RUN vs CMD commands

`RUN` lets us execute commands inside our Docker image. These commands get executed once at build time. For instance if we want to install a package inside a specific directory for our Docker image, then we can use `RUN` command. For example: `RUN mkdir -p /path/to/folder`

`CMD` lets us define a default command to run when container starts.

## Dockerfile Example


```dockerfile
# Use the official image as a parent image.
FROM node:current-slim

# Set the working directory.
WORKDIR /usr/src/app

# Copy the file from your host to container current location (/usr/src/app).
COPY package.json .

# Run the command inside your image filesystem.
RUN npm install

# Add metadata to the image to describe which port the container is listening on at runtime.
EXPOSE 8080

# Run the specified command within the container.
CMD [ "npm", "start" ]

# Copy the rest of your app's source code from your host to your image filesystem.
COPY . .
```

This file takes the following steps:

- Start `FROM` the pre-existing `node:current-slim` image. This is an *official image*, built by the node.js vendors and validated by Docker to be a  high-quality image containing the Node.js Long Term Support (LTS)  interpreter and basic dependencies.
- Use `WORKDIR` to specify that all subsequent actions should be taken from the directory `/usr/src/app` *in your image filesystem* (never the host’s filesystem).
- `COPY` the file `package.json` from your host to the present location (`.`) in your image (so in this case, to `/usr/src/app/package.json`)
- `RUN` the command `npm install` inside your image filesystem (which will read `package.json` to determine your app’s node dependencies, and install them)
- `COPY` in the rest of your app’s source code from your host to your image filesystem.
- The `CMD` directive is the  example of specifying some metadata in your image that describes how to  run a container based on this image. In this case, it’s saying that the  containerized process that this image is meant to support is `npm start`.
- The `EXPOSE 8080` informs Docker that the container is listening on port 8080 at runtime.

That's it wait for the next one ...


