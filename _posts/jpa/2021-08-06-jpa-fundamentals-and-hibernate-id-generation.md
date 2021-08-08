---
layout: post
title: "JPA Fundamentals & Hibernate - 2) @Id generation"
date: 2021-08-06 12:45:31 +0530
categories: "jpa"
author: "mehmetozanguven"
---

In this post, we are going to find out **"what are the options to provide automatic Id generation instead of setting manually". We will look at the generation strategies.**

To use generation strategies, we should use `@GeneratedValue` annotation which has two properties:

- strategy: define generation strategy
- generator: id generator

Topics are:

- [**Github Link**](#github_link)
- [**GeneratedValue.Strategy**](#generate_strategy)
- [**Generation Type - Table**](#generation_type_table)
  - [**Overwrite the default column names**](#overwrite_col_names)
- [**Generation Type - Identity**](#generation_type_identity)
- [**Generic Generator - When Integer/Long generator is not enough**](#generic_generator)

## Github Link <a name="github_link"></a>

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/jpa_fundamentals_and_hibernate/tree/master/id-generation)

## GeneratedValue.Strategy <a name="generate_strategy"></a>

We have four strategies: (represent as enum values)

- `GenerationType.AUTO` : If we use this type, we allow the persistence provider(in our example it is the Hibernate) to choose the appropriate strategy.

> If you use Hibernate, Hibernate will choose the appropriate strategy depending on the database. If you use MySQL, hibernate will choose different strategy rather than PostgreSQL

- `GenerationType.IDENTITY`: If we use this type, we already set the auto-increment feature when we were creating our sql table.
- `GenerationType.TABLE`: If we use this type, there is another table which is used as a generator.
- `GenerationType.SEQUENCE`: Similar to `TABLE`, difference is that implementation will use the sequence in the sql.

I will only talk about the `GenerationType.TABLE` and `Generation.IDENTITY`. The rest is up to you.

## Generation Type - Table <a name="generation_type_table"></a>

In this case, we need separate table, id generation. By default in the JPA, Id generation table should have two columns namely **sequence_name with type VARCHAR(100)** & **next_val with type INT**.

In the `@GeneratedValue`, we will give the id generation table name.

Let's do it an example:

- First drop the product table and re-create:

```sql
DROP TABLE product;

CREATE TABLE product
(
	id INT,
    name VARCHAR(100),
    price NUMERIC,
    expiration_date DATE
);
```

- Then create id generation table:

```sql
CREATE TABLE id_generation
(
	sequence_name VARCHAR(100),
	next_val INT
);
```

- After that update the strategy on the entity:

```java
@Entity
@Table(name = "product")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.TABLE, generator = "id_generation")
    private int id;
    // ...
}
```

- Finally run the main class:

```java
public class Main {
    public static void main(String[] args) {
  		// ...
        Product product = new Product();
        product.setName("Cheese");
        product.setPrice(5.4);
        product.setExpirationDate(LocalDate.now());

        try {
            entityManager.getTransaction().begin();
            entityManager.persist(product);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            System.out.println("Exception: " + e.getMessage());
        }
    }
}
```

First let's look at the tables:

```sql
select * from product ;
 id |  name  | price | expiration_date
----+--------+-------+-----------------
  1 | Cheese |   5.4 | 2021-08-01
(1 row)

select * from id_generation ;
 sequence_name | next_val
---------------+----------
 product       |      100
(1 row)

```

- In default sequence name will be the class name of the entity

In the application console, there will be many queries run by the hibernate:

```wiki
Hibernate:
    select
        tbl.next_val
    from
        id_generation tbl
    where
        tbl.sequence_name=? for update
            of tbl
Hibernate:
    insert
    into
        id_generation
        (sequence_name, next_val)
    values
        (?,?)
Hibernate:
    update
        id_generation
    set
        next_val=?
    where
        next_val=?
        and sequence_name=?
Hibernate:
    select
        tbl.next_val
    from
        id_generation tbl
    where
        tbl.sequence_name=? for update
            of tbl
Hibernate:
    update
        id_generation
    set
        next_val=?
    where
        next_val=?
        and sequence_name=?
Hibernate:
    insert
    into
        product
        (expiration_date, name, price, id)
    values
        (?, ?, ?, ?)

Process finished with exit code 0
```

### Overwrite the default column names <a name="overwrite_col_names"></a>

If you want to use different column names rather than `sequence_name` & `next_val`, you can use the `@TableGenerator` annotation.

## Generation Type - Identity <a name="generation_type_identity"></a>

- First we need to re-create product table and also indicate auto increment column.

> In the PostgreSQL, `SERIAL` pseudo-type is used to define auto-increment columns in tables.
>
> Actually in the PostgreSQL, a **sequence** is a special kind of database object that generates a sequence of integers. When creating a new table, the sequence can be created through the `SERIAL` pseudo-type.
>
> ```sql
> CREATE TABLE table_name(
>     id SERIAL
> );
>
> -- serial and also primary key
> CREATE TABLE table_name(
>     id SERIAL PRIMARY KEY,
> );
> ```

Here is the table creation query:

```sql
CREATE TABLE product
(
	id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    price NUMERIC,
    expiration_date DATE
);
```

```bash
testdatabase=# drop table product ;
testdatabase=# CREATE TABLE product
testdatabase-# (
testdatabase(# id SERIAL PRIMARY KEY,
testdatabase(#     name VARCHAR(100),
testdatabase(#     price NUMERIC,
testdatabase(#     expiration_date DATE
testdatabase(# );
```

- Then we need to add identity generation strategy to the entity class:

> In real word, type of id field generally is the long

```java
@Entity
@Table(name = "product")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private int id;
	// ...
}
```

- The rest is the save new entity:
  > In this case, I don't need to call setter of id field, this will be done by automatically

```java
public static void main(String[] args) {
        EntityManagerFactory entityManagerFactory = Persistence.createEntityManagerFactory(PERSISTENCE_UNIT);
        EntityManager entityManager = entityManagerFactory.createEntityManager();

        Product product = new Product();
        product.setName("Cheese");
        product.setPrice(5.4);
        product.setExpirationDate(LocalDate.now());

        try {
            entityManager.getTransaction().begin();
            entityManager.persist(product);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            System.out.println("Exception: " + e.getMessage());
        }
    }
```

- Just run the project two times, and look at the table:

```sql
testdatabase=# select * from product ;
 id |  name  | price | expiration_date
----+--------+-------+-----------------
  1 | Cheese |   5.4 | 2021-08-01
  2 | Cheese |   5.4 | 2021-08-01
(2 rows)
```

## Generic Generator - When Integer generator is not enough <a name="generic_generator"></a>

If integer generator is not enough, you should create random string for each id field. You can do that with `this.id = UUID.randomUUID().toString();` but this approach is not recommended. Instead you can use Hibernate specific implementation.

> Because we will use hibernate specific implementation, if you change JPA implementation (such as from Hibernate to EclipseLink) in the future, this method will not work.

Let's do that:

- First drop and re-create product table:

> In this case, id field will be varchar

```sql
DROP TABLE product ;
CREATE TABLE product
(
	id VARCHAR(100),
    name VARCHAR(100),
    price NUMERIC,
    expiration_date DATE
);
```

- Then update the entity:

```java
import org.hibernate.annotations.GenericGenerator; // Hibernate specific implementation

import javax.persistence.*;
import java.time.LocalDate;

@Entity
@Table(name = "product")
public class Product {

    @Id
    @GenericGenerator(name = "uuid", strategy = "org.hibernate.id.UUIDHexGenerator")
    @GeneratedValue(generator = "uuid")
    private String id;
    // ....
}
```

- Finally run the main method and look at the table:

```sql
select * from product ;
                id                |  name  | price | expiration_date
----------------------------------+--------+-------+-----------------
 ff8080817b01e5a6017b01e5a8f30000 | Cheese |   5.4 | 2021-08-01
(1 row)
```

Last but not least, wait for the next post ...
