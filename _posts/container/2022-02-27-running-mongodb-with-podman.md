---
layout: post
title: Running MongoDB with Podman
date: 2022-02-27 00:45:31 +0530
categories: "container"
author: "mehmetozanguven"
---

Instead of running MongoDB locally, we can easily run with Podman. Here are the basic steps you should follow:

## Pull the latest images

```bash
podman pull mongo

podman image ls

REPOSITORY                     TAG         IMAGE ID      CREATED       SIZE
docker.io/library/mongo        latest      dfda7a2cf273  6 days ago    697 MB
```

## Create directory for data

Create a directory to store MongoDB data so that in case you delete the container you will still get your data intact.

```bash
sudo mkdir data
cd data

# location
$ pwd
$ /home/mehmetozanguven/mongodb_docker/data
```

## Create MongoDB instance without authentication

First create new mongodb instance without authentication:

```bash
podman run -dt --name my_mongo -p 27017:27017 -v '/home/mehmetozanguven/mongodb_docker/data:/data/db:Z' docker.io/library/mongo:latest
```

- `--name`: describe the name of the project

- `-d`: run your container in detached mode i.e in the background

- `-p 27017:27017`: map port 27017 of the host to port 27017 on the container.

- `-v` (volume): create persisting data generated and used by podman containers.

## Connect mongodb container and create root user

```bash
podman exec -it my_mongo bash

> show dbs;
admin   0.000GB
config  0.000GB
local   0.000GB
> use admin

> db.createUser(
 {
    user:"root",
    pwd: "1234",
    roles: ["root"]
 }
);
Successfully added user: { "user" : "root", "roles" : [ "root" ] }
```

## Remove container and create again with authentication

```bash

podman ps
87ce0fe422ba  docker.io/library/mongo:latest     --auth      6 minutes ago  Up 6 minutes ago  0.0.0.0:27017->27017/tcp  my_mongo

podman stop my_mongo
podman rm my_mongo
podman run -dt --name data_istanbul -p 27017:27017 -v '/home/mehmetozanguven/mongodb_docker/data:/data/db:Z' docker.io/library/mongo:latest --auth


```

After all you can connect your container from your spring project (in the IDE - such as IntelliJ, Eclipse - )

```properties
spring.data.mongodb.authentication-database=admin
spring.data.mongodb.username=root
spring.data.mongodb.password=1234
spring.data.mongodb.database=my_mongo
spring.data.mongodb.port=27017
spring.data.mongodb.host=localhost
```
