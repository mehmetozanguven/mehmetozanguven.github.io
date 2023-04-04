---
layout: post
title: "Check Sessions and Disk Storage in the PostgreSQL"
date: 2021-08-01 12:45:31 +0530
categories: "others"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/check-sessions-and-disk-storage-in-postgresql/"
---

## Check Session

- If you want to list the current session for your database(s), you can use the following sql query:

```sql
SELECT *
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
AND datname = 'database_name';
```

For instance, when I connect the database via Spring Boot application, this sql query returns:

```wiki
...
datid            | 16384
datname          | testdatabase
pid              | 7849
usesysid         | 10
usename          | postgres
application_name | PostgreSQL JDBC Driver
client_addr      | 127.0.0.1
client_hostname  |
client_port      | 49008
...
```

- If you want to list and delete the sessions, you can use the following sql query:

```sql
SELECT *, pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
AND datname = 'database_name';
```

## Check Disk Storage

This is pretty easy, just run the following command:

```sql
SELECT pg_size_pretty( pg_database_size('db_name_or_table_name') );

-- example:
SELECT pg_size_pretty( pg_database_size('testdatabase') );
Returns: 9793 kB
```
