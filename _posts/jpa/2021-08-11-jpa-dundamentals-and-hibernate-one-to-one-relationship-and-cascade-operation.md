---
layout: post
title: "JPA Fundamentals & Hibernate - 6) One To One Relationship & Cascade Operation"
date: 2021-08-11 19:45:31 +0530
categories: "jpa"
author: "mehmetozanguven"
---

In this article, we are going to learn **how to construct a one-to-one relationship between entities in the JPA and when to use Cascade operation**

Topics are:

- [**Github Link**](#github_link)
- [**SQL Setup**](#sql_setup)
- [**One-To-One relationship with `@SecondaryTable` annotation**](#one_to_one_with_secondary_table)
- [**One-to-one relationship with `@OneToOne` annotation**](#one_to_one_with_annotation)
  - [**Uni-Directional One-To-One**](#unidirectional_one_to_one)
  - [**Why we did call persist two times?**](#two_persist_calls)
    - [**Cascade.Persist**](#cascade_persist)
  - [**How to change Foreign Key Name**](#change_foreign_key_name)
  - [**Bi-Directional One-To-One**](#bidirectional_one_to_one)
- [**Fetch Type**](#fetch_type)

## Github Link <a name="github_link"></a>

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/jpa_fundamentals_and_hibernate/tree/master/one-to-one-relation)

## SQL Setup <a name="sql_setup"></a>

We will have two tables `company` and `address` (Actually `address` should be an embeddable object instead of separate table, but this is just an example to understand one-to-one relationship)

```sql
CREATE table company
(
	id SERIAL PRIMARY KEY,
	name VARCHAR(100)
);

CREATE table address
(
    id SERIAL PRIMARY KEY,
    number VARCHAR(100),
	street VARCHAR(100),
	city VARCHAR(100),
    company INT
);
```

## One-To-One relationship with `@SecondaryTable` annotation <a name="one_to_one_with_secondary_table"></a>

In generally, you will use `@OneToOne` annotation to construct the relationship. But there is another way to construct one to one relation even without creating Address entity class using `@SecondaryTable` annotation.

What we should do:

- Create a Company entity with the fields name (fields must include both the Company and Address table's columns)
- Annotate company with `@SecondaryTable(name = "tableName")` to indicate we are using the secondary table
- Annotate fields from the `Address` table using `@Column(name = "tableName")`

```java
@Entity
@Table(name = "company")
@SecondaryTable(name = "Address")
public class Company {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;
    // don't specify the @Column
    // because JPA will know that it is belong to the primary table(company)
    private String name;
    @Column(table = "Address")
    private String street;
    @Column(table = "Address")
    private String number;
    @Column(table = "Address")
    private String city;
    // getters and setters..
}
```

After you persist a company:

```java
public class Main {

    public static void main(String[] args) {
        Company company = new Company();
        company.setName("AB");
        company.setNumber("12");
        company.setCity("İstanbul");
        company.setStreet("Taksim");

        try {
            entityManager.getTransaction().begin();
            entityManager.persist(company);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            System.out.println("Exception: " + e.getMessage());
        } finally {
            entityManager.close();
        }
    }
}
```

In the tables:

```sql
select * from company;
 id | name
----+------
  1 | AB
(1 row)

select * from address;
 id | number | street |   city   | company
----+--------+--------+----------+---------
  1 | 12     | Taksim | İstanbul |
(1 row)
```

We have problem about relationship between Company and Address. At the beginning we said that "relationship between Company and Address must be done via company(foreign key) column in the Address table". In other words, company column in the Address table should have the value of 1 (company-Id).

We should someway tell the JPA to construct relationship with company column in the Address, not the id field (in the Address table). We can do this by using `pkJoinColumns` in the `SecondaryTable` annotation:

> `pkJoinColumns`: are used to join with the primary table (in our case join with the company table)
>
> `@PrimaryKeyJoinColumn`: Specifies a primary key column that is used as a foreign key to join to another table.

```java
@SecondaryTable(name = "Address",
pkJoinColumns = @PrimaryKeyJoinColumn(name = "company"))
public class Company {
    // ...
}
```

After re-create the tables, just run the previous example again:

```sql
select * from company;
 id | name
----+------
  1 | AB
(1 row)
select * from address;
 id | number | street |   city   | company
----+--------+--------+----------+---------
  1 | 12     | Taksim | İstanbul |       1
(1 row)
```

You can add two or more secondary tables.

## One-to-one relationship with `@OneToOne` annotation <a name="one_to_one_with_annotation"></a>

Let's create another two tables called `product` and `detail`

```sql
CREATE TABLE product
(
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE detail
(
    id SERIAL PRIMARY KEY,
    description VARCHAR(100),
    product_id INT
);
```

> For the naming of the foreign key JPA by default expects column with the following format:
>
> - name of the table + "\_" + primary key of the table
> - If you want to change foreign key column name use the `@JoinColumn(name = "col-name")` (but the joinColumn must always on the side of the relationship with the foreign key)

Relationship in the JPA can be represented in two different ways:

- Unidirectional(or one-directional) way
  - Only one class knowns about the other
- Bidirectional way

### Uni-Directional One-To-One <a name="unidirectional_one_to_one"></a>

In that case, class which has a foreign key will know the other class. (In our case it is Detail Class)

> In other words, Detail is the owner of the relationship

Update the detail class:

```java
@Entity
@Table(name = "detail")
public class Detail {
	// ...
    @OneToOne
    private Product product;
}
```

Let's run the main method:

```java
public class Main {
   public static void main(String[] args) {
       Product product = new Product();
       product.setName("Test");

       Detail detail = new Detail();
       detail.setDescription("Test description");
       detail.setProduct(product);

       try {
           entityManager.getTransaction().begin();
           entityManager.persist(product);
           entityManager.persist(detail);
           entityManager.getTransaction().commit();
       }catch (Exception e) {
           System.out.println("Exception: " + e.getMessage());
       } finally {
           entityManager.close();
       }
   }
}
```

After that run the sql queries:

```sql
testdatabase=# select * from product;
 id | name
----+------
  1 | Test
(1 row)

testdatabase=# select * from detail;
 id |   description    | product_id
----+------------------+------------
  1 | Test description |          1
(1 row)
```

### Why we did call persist two times? <a name="two_persist_calls"></a>

It is possible to persist only detail after setting the product? Like this:

```java
// ...
entityManager.getTransaction().begin();
// entityManager.persist(product);
entityManager.persist(detail);
entityManager.getTransaction().commit();

```

**The answer is no, because Product we are trying to refer will not be part of the context.** However JPA provides an alternative solution to this problem. If you want to add Product to the context without calling `persist` , you should use `Cascade` attribution on the `@OneToOne` annotation.

#### Cascade.Persist <a name="cascade_persist"></a>

Update the Detail class and comment the `entityManager.persist(product);` :

```java
@Entity
@Table(name = "detail")
public class Detail {
    // persist product also, when we call persist(detail)
    @OneToOne(cascade = CascadeType.PERSIST)
    private Product product;
}
```

### How to change Foreign Key Name <a name="change_foreign_key_name"></a>

If we should have created the table like this, JPA couldn't have found the foreign key in the detail table, because it wouldn't be matched with default style:

```sql
CREATE TABLE product
(
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE detail
(
    id SERIAL PRIMARY KEY,
    description VARCHAR(100),
    my_custom_product_foreign_key_column INT
);
```

To match with custom foreign key column name:

- use the `@JoinColumn(name = "col-name")`
- **but the joinColumn must always on the side of the relationship with the foreign key**

```java
@Entity
@Table(name = "detail")
public class Detail {
	// ...
    @JoinColumn(name = "my_custom_product_foreign_key_column")
    @OneToOne(cascade = CascadeType.PERSIST)
    private Product product;
}
```

### Bi-Directional One-To-One <a name="bidirectional_one_to_one"></a>

If we want o implement bi-directional:

- we need to create a detail field in the other side of the owner ship.
- We need to use `@OneToOne(mappedBy = "fieldNameOnTheOwnerSide")` annotation
- We need to set detail in the product class as well, if we have bi-directional relationship

```java
public class Detail {
    @JoinColumn(name = "product_id")
    @OneToOne(cascade = CascadeType.PERSIST)
    private Product product; // field name on the owner side
}

@Entity
@Table(name = "product")
public class Product {
    @OneToOne(mappedBy = "product")
    private Detail detail;

    // getters and setters..
}
```

Here is the main class:

```java
public class Main {
    public static void main(String[] args) {
        Product product = new Product();
        product.setName("Test");

        Detail detail = new Detail();
        detail.setDescription("Test description");
        detail.setProduct(product);

        // bi-directional setup
        product.setDetail(detail);

        try {
            entityManager.getTransaction().begin();
//            entityManager.persist(product);
            entityManager.persist(detail);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            System.out.println("Exception: " + e.getMessage());
        } finally {
            entityManager.close();
        }
    }
}
```

## Fetch Type <a name="fetch_type"></a>

A JPA association can be **fetched lazily or eagerly**. The fetching strategy is controlled via the fetch attribute of the `@OneToMany , @OneToOne , @ManyToOne , or @ManyToMany`

The fetch attribute can be either `FetchType.LAZY` or `FetchType.EAGER`. By default, `@OneToMany` and `@ManyToMany` associations use the `FetchType.LAZY` strategy while the `@OneToOne` and `@ManyToOne` use the `FetchType.EAGER` strategy instead.

If we leave the default option and try to load one product from the database, jpa will return the detail of the product with `left outer join`:

```java
entityManager.getTransaction().begin();
Product product = entityManager.find(Product.class, 2L);
entityManager.getTransaction().commit();
```

In the console, hibernate will run the following query:

```sql
    select
        product0_.id as id1_3_0_,
        product0_.name as name2_3_0_,
        detail1_.id as id1_2_1_,
        detail1_.description as descript2_2_1_,
        detail1_.product_id as product_3_2_1_
    from
        product product0_
    left outer join
        detail detail1_
            on product0_.id = detail1_.product_id
    where
        product0_.id=?
```

If we change to the `LAZY` and the run the main method:

```java
@Entity
@Table(name = "product")
public class Product {

    @OneToOne(mappedBy = "product", fetch = FetchType.LAZY)
    private Detail detail;
}
```

In the console, hibernate will run the following query:

```sql
select
        product0_.id as id1_3_0_,
        product0_.name as name2_3_0_
    from
        product product0_
    where
        product0_.id=?
```

Last but not least, wait for the next one ...
