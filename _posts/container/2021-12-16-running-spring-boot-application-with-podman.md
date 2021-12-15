---
layout: post
title: Running Spring Boot Application with Podman
date: 2021-12-16 00:45:31 +0530
categories: "container"
author: "mehmetozanguven"
---

To run spring boot application in a container. We should do the following steps:

- Create a jar package with maven or gradle (`mvn clean install` or `mvn clean package`)
- Create a Dockerfile (to containerize an application)
- Build to Dockerfile to create an docker image
- Run the docker image

Finally we will look at the **how to connect running postgres container from Spring Boot application**

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

I will skip the first step: "Create a jar package". Assuming that you are already familiar with.

## Create Dockerfile

We should set the base image for our java application. I will use openjdk version 11. (You may look at the all base images related to the java via [https://hub.docker.com/\_/openjdk](https://hub.docker.com/_/openjdk) )

```dockerfile
FROM openjdk:11
COPY target/yourJarNameInLocal.jar yourJarNameInDocker.jar
ENTRYPOINT ["java","-jar","/yourJarNameInDocker.jar"]
```

## Build docker file

Go to your Dockerfile and run the command

```bash
$ podman build -t yourName .
...
...
COMMIT yourName
--> ea58ea6b407
Successfully tagged localhost/yourName:latest
ea58ea6b40766c62afad735d5b6918a214c6491cf56ecec81df2783133d13c1

# list the images

$ podman images
REPOSITORY                     TAG         IMAGE ID      CREATED             SIZE
localhost/yourName             latest      ea58ea6b4076  About a minute ago  719 MB
docker.io/library/openjdk      11          e273ff3d8df8  5 days ago          671 MB
docker.io/library/httpd        latest      d54056386fbb  5 days ago          142 MB
docker.io/library/postgres     latest      d191afba1bb1  5 days ago          382 MB
docker.io/library/hello-world  latest      feb5d9fea6a5  3 weeks ago         19.9 kB
```

## Run the docker image

```bash
podman run localhost/yourName -dt -p 8000:8080
```

> Because I didn't specify `server.port`, spring application will run on the port 8080. And I am redirecting the requests on localhost:8000 to the localhost/yourName container on port 8080

That's it. Spring application will run inside the container. If we look at the container itself:

```bash
$ podman exec -it {containerId or containerName} bash
root@10959577521a:/# ls
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var	yourJarNameInDocker.jar
```

## Container Spring Boot and PostgreSQL

Let's assume that we should also need to connect postgresql and also I am assuming that we have another container for postgres. At the end SpringBoot container should connect to the Postgres container.

> Finally I am also assuming that Postgres container is already running and there is a port binding such as `-p 5432:5432`

We have three options to connect Postgres container:

- Connect from your IDE (for instance, run button in the IntelliJ IDEA)
- Connect from jar file using the ip address of the postgres host.
- Connect from jar file using shared network between containers

### Connect from IDE to Postgres Container

There is a running postgres container and you want to connect from your IDE (or in other words, you want to connect from your host)

> Host means that "server or laptop that container runs on it"

In this case you don't need to change `spring.datasource.url` property because you will connect the postgres container from the host itself.

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/{yourDbName}
```

### Connect from jar file using ip address of the postgres' host

Learn your host ip address via `$ ip addr` or `$ ifconfig`. Or simply run the following command:

```bash
$ hostname -I | awk '{print $1}'
192.168.122.1
```

If you run two containers on the same of host, ip address should be `192.168...`. For instance , when I want to connect postgres container from my laptop (also I am running spring boot application on my laptop), ip address was the `192.168.122.1`. Finally change the property to:

```properties
spring.datasource.url=jdbc:postgresql://192.168.122.1:5432/{yourDbName}
```

If postgres container is placed on another server, there must be firewall permission also.

To test connection from your laptop, you can use `nc` command:

```bash
[mehmetozanguven@fedora demo]$ nc -vz 192.168.122.1 5432
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Connected to 192.168.122.1:5432.
Ncat: 0 bytes sent, 0 bytes received in 0.01 seconds.
```

### Connect from jar file using shared network between containers

This can only be done by rootful containers. (Basically run every command `podman run`, `podman network` with sudo prefix)

I am not going into detail of this. But there are awesome blog for that (and also for other network communication). Here is the [link](https://www.redhat.com/sysadmin/container-networking-podman)
