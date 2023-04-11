---
layout: post
title: Podman Tutorial
date: 2021-12-15 13:45:31 +0530
categories: "container"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/getting-started-podman/"
---

In this one-shot tutorial, we are going to learn what Podman is, how to use it, the differences between Docker and Podman and more..

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

## What is Podman?

Podman is a daemonless container engine for developing, managing and running container and container image on our linux system.

Podman also provides a Docker-compatible command line and works well with the Docker. In simply, we can also create an alias `alias docker=podman`.

One of the best features of podman is run rootless containers. A rootless container is running and managing containers without root privileges.

**I don't want to bother you with the installation steps. Because podman web page has awesome step by step installation processes for various linux distros, mac and more.. Here is the [link](https://podman.io/getting-started/installation)**

## Podman version

Use the `podman version`command:

```bash
$ podman version
Version:      3.4.2
API Version:  3.4.2
Go Version:   go1.16.8
OS/Arch:      linux/amd64
```

## Podman System information

We can look up the system information via `podman info`command:

```bash
$ podman info
host:
  arch: amd64
  buildahVersion: 1.23.1
  cgroupControllers:
  - memory
  - pids
...
```

## Where podman pull images

At the time of writing this article, podman will look at the following registries to find appropriate image(s):

```wiki
["registry.fedoraproject.org", "registry.access.redhat.com", "docker.io", "quay.io"]
```

These registries are defined in the file : `/etc/containers/registries.conf`

```bash
$ less /etc/containers/registries.conf
# # An array of host[:port] registries to try when pulling an unqualified image, in order.
unqualified-search-registries = ["registry.fedoraproject.org", "registry.access.redhat.com", "docker.io", "quay.io"]

```

## Container Storage

In podman, each user has its own container storage. For instance, if user_a has the image called postgres, then if user_b needs the same image podman will try to pull image from the remote repository instead of using the local image.

Other than root, containers are stored in the directory: `$HOME/.local/share/containers/storage/`

If you want to learn where my container storage is, you can run the `$ podman info` command:

```bash
$ podman info
...
store:
  configFile: /home/mehmetozanguven/.config/containers/storage.conf
  containerStore:
    number: 11
    paused: 0
    running: 0
    stopped: 11
  graphDriverName: overlay
  graphOptions: {}
  graphRoot: /home/mehmetozanguven/.local/share/containers/storage
  graphStatus:
    Backing Filesystem: extfs
    Native Overlay Diff: "true"
    Supports d_type: "true"
    Using metacopy: "false"
  imageStore:
    number: 17
  runRoot: /run/user/1000/containers
  volumePath: /home/mehmetozanguven/.local/share/containers/storage/volumes
```

Let's continue with the example,

## Example: Httpd Container

We can pull httpd images from the docker repo:

```bash
[mehmetozanguven@fedora ~]$ podman pull docker.io/library/httpd

[mehmetozanguven@fedora ~]$ podman pull docker.io/library/httpd
Trying to pull docker.io/library/httpd:latest...
Getting image source signatures
Copying blob d0c6942edac3 done
Writing manifest to image destination
Storing signatures
d54056386fbb1ea69f9332f35ab083dd7062cc9cb78ed28ce6b8f85e9dfb56b3
```

We can verify the images by listing all images:

```bash
 [mehmetozanguven@fedora ~]$ podman images
 REPOSITORY                     TAG         IMAGE ID      CREATED      SIZE
 docker.io/library/httpd        latest      d54056386fbb  2 days ago   142 MB
```

To run httpd container

```bash
$ podman run -d -p 8080:80 d54056386fbb
bd0ccd21901684b7e304fdc14194889077e4095b12708f28b9e2d50103cf5e02
```

- `-d` means => runs in detached mode
- `-p 8080:80`: Run Container in port 80 AND
  - For all requests in the host with port 8080 redirects to the container port 80

Now go to the `http://localhost:8080/`, you will see=> **It works!**

> You can also access port 8080 from your mobile phone, tablet etc. But these devices also must connect to the same network (Basically same wifi).
>
> For instance my local host ip address (the one running the podman) `192.168.1.16`, if i open the `http://196.186.1.16:8080`from my mobile browser, i will see that: **It works!**

To see running container:

```bash
[mehmetozanguven@fedora ~]$ podman ps -a
CONTAINER ID  IMAGE                           COMMAND           CREATED        STATUS            PORTS                 NAMES
bd0ccd219016  docker.io/library/httpd:latest  httpd-foreground  4 minutes ago  Up 4 minutes ago  0.0.0.0:8080->80/tcp  infallible_shockley
```

To get container's pid number:

```bash
$ podman top -l
# or
$ podman top -{containerId}
```

> `-l` is used for **latest container**

To stop container:

```bash
$ podman stop -l
# or
$ podman stop {containerId}
```

To remove container:

```bash
$ podman rm -l
# or
$ podman rm {containerId}
```

To view container's log:

```bash
$ podman logs -l
```

That's the basic instructions for podman, you can find more and more on the Internet. I will continue with the postgresql example.
