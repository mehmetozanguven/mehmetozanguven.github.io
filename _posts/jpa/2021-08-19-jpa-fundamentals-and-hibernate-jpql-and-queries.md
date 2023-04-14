---
layout: post
title: "JPA Fundamentals & Hibernate - 12) JPQL and Queries"
date: 2021-08-19 19:45:31 +0530
categories: "jpa"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/jpa/jpa-fundamentals-and-hibernate-jpql-and-queries/"
---

In this article, we will learn **what is the JPQL(The Java Persistence Query Language)** and **how to use it**.

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

## What is the JPQL

The Java Persistence query language (JPQL) is used to define searches against persistent entities independent of the mechanism used to store those entities. In general, using JPQL we can write only one SQL-Like query that will work for both MySQL, PostgreSQL, MSSQL etc..

JPQL is "portable", and not constrained to any particular data store.

Let's assume that we have the following table(s) in the database:

```sql
CREATE TABLE product
(
    id INT,
    name VARCHAR(100),
    cost NUMERIC
);
INSERT INTO product (id, name, cost) VALUES (1, 'P1', 10);
INSERT INTO product (id, name, cost) VALUES (2, 'P2', 15);
INSERT INTO product (id, name, cost) VALUES (3, 'P3', 5);
INSERT INTO product (id, name, cost) VALUES (4, 'P4', 20);
```

And we have a product entity:

```java
@Entity
@Table(name = "product")
public class Product {
    @Id
    private long id;
   	private String name;
    @Column(name = "cost")
    private double price;
}
```

## How to select all the records

The difference between in the JPQL and SQL is that we select from the entity if we are using the JPQL. That's the reason why we should use `Product` (entity name) not the `product` (table name):

- SQL version:

```sql
SELECT * FROM product p; --native query
```

- JPQL version:

```sql
SELECT p FROM Product p;
```

```java
public class Main {
    public static void main(String[] args) {
        try {
            entityManager.getTransaction().begin();

            String selectAll = "SELECT p FROM Product p";
            TypedQuery<Product> sqlResult = entityManager.createQuery(selectAll, Product.class);
            List<Product> products = sqlResult.getResultList();

            entityManager.getTransaction().commit();
        }catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }
    }
}
```

## How to use parameters (where statement)

Don't write queries for the database column(s), use the field column(s) of the entity

- SQL version:

```sql
SELECT * FROM product p WHERE p.cost = 10;
```

- JPQL version:

```sql
SELECT p FROM Product p WHERE p.price = 10;
```

- To add parameter use the semicolon followed by the parameter's name => `:parameterName`

> Do not add whitespace between semicolon and parameter's name

```java
public class Main {
    public static void main(String[] args) {
        try {
            entityManager.getTransaction().begin();

            String whereSelect = "SELECT p FROM Product p WHERE p.price > :myPriceParamName";
            TypedQuery<Product> sqlResult = entityManager.createQuery(whereSelect, Product.class);
            sqlResult.setParameter("myPriceParamName", 10.0);
            List<Product> products = sqlResult.getResultList();

            entityManager.getTransaction().commit();
        }catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }
    }
}
```

## How to use aggregate function

Change the return type:

```java
public class Main {
    public static void main(String[] args) {
        try {
            entityManager.getTransaction().begin();

            String aggregateQuery = "SELECT SUM(p.price) FROM Product p WHERE p.price > :myPriceParamName";
            TypedQuery<Double> sqlResult = entityManager.createQuery(aggregateQuery, Double.class);
            sqlResult.setParameter("myPriceParamName", 10.0);
            double sum = sqlResult.getSingleResult();

            entityManager.getTransaction().commit();
        }catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }
    }
}
```

## @NamedQuery

Instead of directly writing JPQL as a string value in our program, we can group all the queries for each entities using the `@NamedQuery`annotation. Basically we will write all the queries about the entity using `@NamedQuery` annotation on the entity itself.

**Even if namedQuery simplifies the readability, it is not used in the production. Because when our sql queries is becoming more complex, it will be hard to maintain @NamedQuery annotation**

NamedQuery will have:

- the name of the query
- JPQL itself

Let's do it an example:

```java
@Entity
@Table(name = "product")
@NamedQuery(name = "Product.all", query =. "SELECT p FROM Product p")
@NamedQuery(name = "Product.findById", query =. "SELECT p FROM Product p WHERE p.id = :id")
public class Product {
    @Id
    private long id;
   	private String name;
    @Column(name = "cost")
    private double price;
}
```

```java
public class Main {
    public static void main(String[] args) {
        try {
            entityManager.getTransaction().begin();

            TypedQuery<Product> allProducts = entityManager.createNamedQuery("Product.all", Product.class);
            TypedQuery<Product> productById = entityManager.createNamedQuery("Product.findById", Product.class);
            productById.setParameter("id", 1L);

            entityManager.getTransaction().commit();
        }catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }
    }
}
```

> Not if you the JPA version <2.2, then it is not possible to add multiple `@NamedQuery`annotations, instead you should group into the one annotation called `@NamedQueries`
>
> ```java
> @NamedQueries(
>   {
>     @NamedQuery(name = "Product.all", query =. "SELECT p FROM Product p"),
>     @NamedQuery(name = "Product.findById", query =. "SELECT p FROM Product p WHERE p.id = :id")
>   }
> )
> public class Product {}
> ```
