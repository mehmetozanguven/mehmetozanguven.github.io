---
layout: post
title: Quick Intro to Container(s)
date: 2021-10-26 13:45:31 +0530
categories: "container"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/quick-intro-to-containers/"
---

Recently, I decided to learn containers (specifically Podman) to simply run my applications in an isolated environment. I can directly jump into the Podman and can run my application. But this would be bad because learning the basic fundamentals are essentials before really diving into it. Before Podman, I just run some applications with Docker (without knowing fundamentals). After that I found Podman was more easy to run and also it is already installed on my linux distro (which is Fedora 34). At the end I decided to learn all related concepts about containers.

And also I though that it is time to write some blogs about container :)

We are using terminologies such as container, container images interchangeably (and more). But there are differences between them. Let's start with the basic question: "what the Container is"

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

## What is the Container ?

> Basically, a container is just a simple Linux process

A container is really two different things and it has two states:

- Rest
- Running (or simply run)

In the **rest** mode, a container is file (or files) which is saved on disk. This is referred to as **Container Images** or **Container Repository**

When we **run** the container, the **Container Engine** takes requires files and meta datas out of the box and gives them to the Linux Kernel. Starting a container is similar to starting a linux process. After running, Containers are just a linux process.

> Container image format on disk are defined by the standards.

Even there are several Container Image formats, industry standard is the **Open Container Initiative(OCI)**.

**Container Engines take the Container Image and turn it into a Container** (running process). Running process protocols are also governed by the OCI.

## What is the Open Container Initiative (OCI) ?

The purpose of the OCI is to encourage people to use a set of common minimal, open standards and specifications around container technology.

In a simple term, OCI purpose is that: "when you run your (containerized) application on MACOS, you sure that it will also work on Windows and Linux"

## Container Image

A container image, is a file which is pulled down from a Registry Server and used locally as a starting point when running Containers.

A container can be composed of one images or more images.

Image(s) format is defined by the OCI.

## Container Engine

It is a software which accepts user requests (for instance it can accept user requests from command line), pulls images and runs the container.

Most Containers rely on an OCI compliant runtime.

## Container

> Containers have existed within operating systems for quite a long time.

A container is the runtime instantiation of a Container Image.

It is standard Linux process.

## Container Host

Container host is the system that runs the containerized process(es)

## Registry Server

It is file server that is used to store container files.

---

At the end we are mostly using containers for our applications such as SpringBoot, Nodejs and more.. to create and run these applications in a isolated environment. This gives guarantee (almost) that these applications also work for the production, test and local environments.

There are tons of other concepts about Container such as Image Layer, Container Runtime, Graph Driver and more.. But I think these information is enough to start with the Podman.
