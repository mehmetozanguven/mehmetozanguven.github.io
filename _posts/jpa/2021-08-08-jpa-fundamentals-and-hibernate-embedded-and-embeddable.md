---
layout: post
title: "JPA Fundamentals & Hibernate - 4) @Embeddable and @Embedded"
date: 2021-08-08 19:45:31 +0530
categories: "jpa"
author: "mehmetozanguven"
---

In this post, we are going to learn **when and how to use embaddable object in Java and JPA**.

Topics are:

- [**Github Link**](#github_link)
- [**When to use Embeddable Object**](#when_to_use)
- [**SQL Setup**](#sql_setup)
- [**JPA Setup**](#jpa_setup)
  - [**Create embeddable class**](#create_embeddable_class)
  - [**Embed class to an entity**](#embed_class)

## Github Link <a name="github_link"></a>

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/jpa_fundamentals_and_hibernate/tree/master/embeddable-and-embedded)

## When to use Embeddable Object <a name="when_to_use"></a>

The main reason to use embeddable object is to split up large entity classes. In the database world, a large table (one with many columns) is okey. Breaking up to this table to many separate tables make things getting worse. On the other, in the software world, we try to have small classes which responsible only one thing. **`@Embedded/@Embeddable` is used when we want to split entity classes into smaller units without creating many separate tables.**

For instance , let's say you have table called `company` and it has the following columns:

- id of type INT
- name of type varchar
- number of type varchar
- street of type varchar
- city of type varchar

This design is okey for database world, but in the software level we could say that there should be two separate classes, one for company (id and only name) and the other for Address (number, street and city). After all we could think that we need to create two tables(`company` and `address`) instead of one.

But `Address` has no value in the database if there is no `Company`, but `Company` can stand in the database even if we have no `Address` information. Objects such as `Address` then can be used as embedded objects in the JPA.

Let's implement this feature.

## SQL Setup <a name="sql_setup"></a>

First create table called `company`

```sql
CREATE table company
(
	id SERIAL PRIMARY KEY,
	name VARCHAR(100),
	number VARCHAR(100),
	street VARCHAR(100),
	city VARCHAR(100)
);
```

## JPA Setup <a name="jpa_setup"></a>

### Create embeddable class <a name="create_embeddable_class"></a>

We need to create an embeddable class (we are saying that this class is part of an entity)

```java
@Embeddable
public class Address {
    private String number;
    private String street;
    private String city;
    // ...
}
```

### Embed class to an entity <a name="embed_class"></a>

The rest is to embed to class to an entity. (using `@Embedded` )

```java
@Entity
@Table(name = "company")
public class Company {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column(name = "name")
    private String name;

    @Embedded
    private Address address;
}
```

But still we have a problem: how to map column with Embeddable objects? We can use `@AttributeOverrides` annotation:

```java
@Entity
@Table(name = "company")
public class Company {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column(name = "name")
    private String name;

    @Embedded
    @AttributeOverrides({
            @AttributeOverride( name = "number", column = @Column(name = "number")),
            @AttributeOverride( name = "street", column = @Column(name = "street")),
            @AttributeOverride( name = "city", column = @Column(name = "city"))
    })
    private Address address;
}
```

Final step just to run the code

### Run the code

```java
public class Main {

    public static void main(String[] args) {
		// ...
        Address address = new Address();
        address.setCity("İstanbul");
        address.setNumber("123");
        address.setStreet("Taksim");

        Company company = new Company();
        company.setName("test");
        company.setAddress(address);

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

After that if we look at the table:

```sql
select * from company;
 id | name | number | street |   city
----+------+--------+--------+----------
  1 | test | 123    | Taksim | İstanbul
(1 row)
```

That's it, wait for the next one ...
