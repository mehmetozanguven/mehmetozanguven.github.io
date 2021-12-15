---
layout: post
title: Running PostgreSQL with Podman
date: 2021-12-15 21:45:31 +0530
categories: "container"
author: "mehmetozanguven"
---

Instead of running postman locally, we can easily run with Podman. Here are the basic steps you should follow.

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

## Search available Postgres Images

You can search via this command:

```bash
$ podman search postgres
INDEX              NAME                                                                    DESCRIPTION                                      STARS       OFFICIAL    AUTOMATED
fedoraproject.org  registry.fedoraproject.org/f31/postgresql                                                                                0
redhat.com         registry.access.redhat.com/cloudforms45/cfme-openshift-app              Red Hat® CloudForms Appliance image to be u...   0
docker.io          docker.io/library/postgres                                              The PostgreSQL object-relational database sy...  9999        [OK]
```

The one we are going to use is the official one called `docker.io/library/postgres`

```bash
$ podman pull docker.io/library/postgres
Trying to pull docker.io/library/postgres:latest..
...
Writing manifest to image destination
Storing signatures
d191afba1bb1ee80e06afbdca13962b6f3ac9900df5e8d3a4a9f06e8188a8bac
```

Run the command `podman images` to verify it is downloaded:

```bash
[mehmetozanguven@fedora ~]$ podman images
REPOSITORY                     TAG         IMAGE ID      CREATED      SIZE
docker.io/library/httpd        latest      d54056386fbb  5 days ago   142 MB
docker.io/library/postgres     latest      d191afba1bb1  5 days ago   382 MB
docker.io/library/hello-world  latest      feb5d9fea6a5  3 weeks ago  19.9 kB
```

## Run the container

Because data inside the Container(s) will be deleted when we remove the container's images, we should map the data created by the Postgres to our local folder. Create a folder that will hold postgres data:

Then run the following command:

```bash
podman run -dt --name my-postgres -e POSTGRES_PASSWORD=1234 -v "/home/mehmetozanguven/postgres_docker:/var/lib/postgresql/data:Z" -p 5432:5432 postgres
```

- `-dt`: run in detach mode
- `--name`: our postgres container name
- `-e` : is used to define environment variable, `POSTGRES_PASSWORD`will be the password for our postgres container
- `-v`: Store data inside the container to path `${HOME}/postgres_docker`
- `-p`: Any request on the port 5432 inside my computer will be redirected to the container my-postgres on the port 5432.
- `:Z` : allows container to write to the volume, but doesn’t allow the volume to be shared with other containers. Basically this is for SELinux configuration

> If you want to see logs also:
>
> ```bash
> $ podman --log-level=debug run -d --name my-postgres -e POSTGRES_PASSWORD=1234 -v "/home/mehmetozanguven/postgres_docker:/var/lib/postgresql/data:Z" -p 5432:5432 postgres
> ```

If you want to run your container as your access level, you should use `--userns=keep-id` parameter. If you don't specify this parameter, you won't be access the folder => `/home/mehmetozanguven/postgres_docker`. This folder only be accessible with root.

```bash
 podman --log-level=debug run -d --userns=keep-id --name my-postgres -e POSTGRES_PASSWORD=1234 -v "/home/mehmetozanguven/postgres_docker:/var/lib/postgresql/data:Z" -p 5432:5432 postgres
```

To verify container is running:

```bash
[mehmetozanguven@fedora ~]$ podman ps
CONTAINER ID  IMAGE                              COMMAND     CREATED        STATUS            PORTS                   NAMES
f9ce4c13c5e0  docker.io/library/postgres:latest  postgres    3 minutes ago  Up 3 minutes ago  0.0.0.0:5432->5432/tcp  my-postgres
```

## Connect the container

```bash
$ podman exec -it my-postgres bash
mehmetozanguven@545739c0d37e: /$
# if you don't set parameter called --userns=keep-id, it will be:
root@545739c0d37e: /$
```

## Use psql utility tool to access PostgreSQL

After connecting to the container:

```bash
root@545739c0d37e:/$ psql -U postgres
psql (14.0 (Debian 14.0-1.pgdg110+1))
Type "help" for help.

postgres=# \l
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres

```

The rest is related to the postgresql itsel. (`CREATE DATABASE myDB`, `CREATE TABLE IF NOT EXISTS ...` etc ..)

## Stop the container

You can use the following command to stop your postgres container:

```bash
$ podman stop my-postgres
```

## Error: error creating container storage: the container name "my-postgres" is already in use by

This means that there is an already container named as `my-postgres`.

First try to find container:

```bash
$ podman ps -a
CONTAINER ID  IMAGE                                 COMMAND           CREATED        STATUS                    PORTS                   NAMES
4c4b4856be2e  docker.io/library/postgres:latest     postgres          5 seconds ago  Exited (1) 5 seconds ago  0.0.0.0:5432->5432/tcp  my-postgres
```

To solve that error you should have two options:

1. Start the container again:

```bash
$ podman start my-postgres
# or by container id
$ podman start 4c4b4856be2e
```

2. Remove the container and create again with `podman run`

```bash
$ podman rm my-postgres
# or by container id
$ podman rm 4c4b4856be2e

$ podman run -d --name my-postgres ...
```

That's the basic steps to install postgresql via podman. In the next article, we will run spring boot and postgresql inside containers
