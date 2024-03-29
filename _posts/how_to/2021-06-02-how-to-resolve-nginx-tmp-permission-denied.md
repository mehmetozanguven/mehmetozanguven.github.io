---
layout: post
title: How to resolve -- /var/lib/nginx/tmp/ Permission Denied Error
date: 2021-06-02 14:45:31 +0530
categories: "how-to"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/how-to/how-to-resolve-nginx-tmp-permission-denied/"
---

Recently I have encountered this error. Problem is that: user that runs nginx service has no permission to access `/var/lib/nginx/tmp` file.

Example:

```wiki
2021/06/02 14:46:27 [crit] 5148#0: *35 open()
"/var/lib/nginx/tmp/proxy/0/01/0000000010" failed (13: Permission denied) while
reading upstream, ...
```

Solution: Find the user that runs the nginx service(You can find in the nginx.conf file). And add permission to that user for the `/var/lib/nginx` folder

```nginx
user yourUserName;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
http {
    ...
}
```

But before doing that we need to globally give permission for **SELinux**

```bash
sudo setenforce 0
```

> Security-Enhanced Linux (SELinux) is a security architecture for linux systems that allows to have more control over who can access the system

```bash
sudo chown -Rf yourUserName:yourUserName /var/lib/nginx
```
