---
layout: post
title: How to resolve -- PR_CONNECT_RESET_ERROR
date: 2021-05-23 14:45:31 +0530
categories: "how-to"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/how-to/how-to-resolve-pr-connect-reset-error/"
---

This error might releted to your Nginx configuration(at least problem was the nginx for me)

After enable the SSL connection and try to connect from the browsers:

- Firefox returns :

```wiki
An error occurred during a connection to example.com PR_CONNECT_RESET_ERROR

    The page you are trying to view cannot be shown because the authenticity of the received data could not be verified.
```

- Google Chrome returns:

```wiki
The connection was reset.
Try:

Checking the connection
Checking the proxy and the firewall
ERR_CONNECTION_RESET
```

In my configuration, I only force the TLSv1.3

```nginx
http {
	ssl_protocols TLSv1.3;
}
```

And nginx version:

```bash
$ nginx -V
nginx version: nginx/1.18.0
built by gcc 7.3.1 20180712 (Red Hat 7.3.1-10) (GCC)
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
```

Problem is that, **OpenSSL 1.0.2k-fips** doesn't support the TLSv1.3, either I should update the openssl or support both TLSv1.2. Quick solution could be just updating nginx.conf file like this:

```nginx
http {
	# if openssl version is 1.0.2 then can not enable tlsv1.3, gives an error PR_CONNECT_RESET_ERROR
	ssl_protocols TLSv1.2 TLSv1.3;
}
```
