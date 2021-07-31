---
layout: post
title: "BackUp PostgreSQL with pg_dump"
date: 2021-07-31 12:45:31 +0530
categories: "others"
author: "mehmetozanguven"
---

In this post, we are going to find out how to backup postgresql database with `pg_dump` utility.

Topics:

[TOC]

## What is the `pg_dump` utility

PostgreSQL provides the `pg_dump` utility to simplify backing up a single database. This command
must be run as a user with read permission to the database we intend to back up.

## Backup File Types

3 types of backup files we have:

- **.bak**: compressed binary format
- **.sql**: plaintext dump
- **.tar**: tarball

## Steps to backup database

- Log in as the `postgres` user:

```bash
$ su - postgres
```

- Dump the contents of a database to a file. Replace dbname with the name of the database to be backed up:

```bash
$ pg_dump your_db_name > backup_file_name.bak
```

- If we want to dump without log as the postgres user:

```bash
$ pg_dump -U postgres -W -h localhost your_db_name > backup_file_name.bak
```

## Remote Database backup

- `pg_dump` can be run from a client computer to back up data on a remote server. Use -h flag
  to specify the IP address and -p to identify the port:

```bash
$ pg_dump -h 13.34.232.23 -p 5432 your_db_name > backup_file_name.bak
# or
$ pg_dump -h 13.34.232.23 -p 5432 your_db_name > backup_file_name.sql
```

## Backup Only the scheme definitions (create tables etc..)

```bash
$ pg_dump -U postgres -W -h localhost your_db_name --schema-only > schemeonly.sql
```

## Backup Only the Data, not the schema

```bash
$ pg_dump -U postgres -W -h localhost your_db_name --data-only > dataOnly.sql
```

## Backup Only One Table (Be Careful when using in the production !!)

```bash
$ pg_dump -U postgres -W -h localhost your_db_name -t your_table_name > tableBackup.sql
```

> Note: When -t is specified, pg_dump makes no attempt to dump any other database
> objects that the selected table(s) might depend upon. Therefore, there is no guarantee that
> the results of a specific-table dump can be successfully restored by themselves into a clean
> database.

## Backup dump data as INSERT commands rather than Copy

```bash
$ pg_dump -U postgres -W -h localhost local_Db --column-inserts --data- only > testAccount.sql
```

Dump data as INSERT commands (rather than COPY ). **This will make restoration very slow**

## How to run .sql file in the postgresql

```bash
$ psql -U postgres -d your_db_name -h localhost -p 5432 -a -f /path/to/sql
## LOCAL DB EXAMPLE
$ psql -U postgres -d local_Db -h localhost -p 5432 -a -f /path/to/sql
```

- `-U` : username
- `-d`: database name
- `-a`: all echo
- `-f`: path to SQL script

## To Load Backup File

- Drop the previous database and create the empty database again:

```sql
DROP DATABASE IF EXISTS databasename;
CREATE DATABASE databasename
```

- Restore the database using psql:

```bash
$ psql databasename < backup_file_name.bak
```
