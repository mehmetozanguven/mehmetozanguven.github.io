---
layout: post
title: "JPA Fundamentals & Hibernate - 10) Inheritance Strategies"
date: 2021-08-15 12:45:31 +0530
categories: "jpa"
author: "mehmetozanguven"
---

In this article, we are going to look **how to represent the inheritance relationship in the JPA** and **how to use `@MapperSuperclass` annotation**

{% include auto_table_of_contents.html %}

## Github Link

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/jpa_fundamentals_and_hibernate/tree/master/inheritance-strategies)

---

We have three inheritance strategies in the JPA and these strategies are defined in the top level class using `@Inheritance` annotation. `@Inheritance` specifies three strategies:

```java
public @interface Inheritance {

    /** The strategy to be used for the entity inheritance hierarchy. */
    InheritanceType strategy() default SINGLE_TABLE;
}

public enum InheritanceType {

    /** A single table per class hierarchy. */
    SINGLE_TABLE,

    /** A table per concrete entity class. */
    TABLE_PER_CLASS,

    /**
     * A strategy in which fields that are specific to a
     * subclass are mapped to a separate table than the fields
     * that are common to the parent class, and a join is
     * performed to instantiate the subclass.
     */
    JOINED
}
```

For all inheritance strategies, when you defined `@Id` in the top level class, you don't need to define again in the lower level classes(which extends top level class).

For this article, I will not touch the `InheritanceType.TABLE_PER_CLASS`. And in general this approach is not recommended in the production level.

## InheritanceType.SINGLE_TABLE

> Image that we have two entites called `Animal` & `Cat` . `Cat` is also extending animal class as well.

In this case, we will have only one table (let's called `animal`), and `animal` table will include the `color` & `dtype`.

`dtype` is used as discriminator for the entities. It basically says that the record belongs to the `Cat` class or `Animal` class or `Dog` class.

Let's run the following sql queries:

```sql
CREATE TABLE animal
(
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    color VARCHAR(100),
    dtype VARCHAR(100)
);
```

Here is the animal entity:

```java
@Entity
@Table(name = "animal")
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
public class Animal {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private String name;
}
```

Cat entity:

```java
@Entity
public class Cat extends Animal {
    private String color;
}
```

Run the main class and look at the tables:

```java
public class Main {
    public static void main(String[] args) {
        Animal animal = new Animal();
        animal.setName("animal");

        Cat cat = new Cat();
        cat.setName("cat");
        cat.setColor("black");

        try {
            entityManager.getTransaction().begin();
            entityManager.persist(animal);
            entityManager.persist(cat);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }
    }
}
```

```sql
select * from animal;
 id |  name  | color | dtype
----+--------+-------+--------
  1 | animal |       | Animal
  2 | cat    | black | Cat
(2 rows)
```

As we can see, color column won't be null only if the class type is the Cat. But `SINGLE_TABLE` approach has some problems:

1. We will have a null values. For instance, after a period of time if we add another class `Dog` which has also specific column different than Animal and Cat, then we will have another null column values (which is hard to maintain)
2. We have to store application level information (class type) in the database. (In the database level, we should never store application level information)

## InheritanceType.JOINED

> Image that we have `Product` (top class) and `Computer(extends Product)` objects.

In this case, JPA will store the fields that are specific to the class to a separate table.

Run the following sql queries:

```sql
CREATE TABLE product
(
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE computer
(
    ram VARCHAR(100),
    id INT -- it is the primary id of the product table to create relationship
);
```

Create the Product top level entity class:

```java
@Entity
@Table(name = "product")
@Inheritance(strategy = InheritanceType.JOINED)
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private String name;
}
```

Create the Computer lower level entity class:

```java
@Entity
@Table(name = "computer")
public class Computer extends Product {
    private String ram;
}
```

Now, run the main class and see the tables:

```java
public class Main {
    public static void main(String[] args) {
        Product product = new Product();
        product.setName("product");

        Computer computer = new Computer();
        computer.setName("Asus");
        computer.setRam("10GB");

        try {
            entityManager.getTransaction().begin();
            entityManager.persist(product);
            entityManager.persist(computer);

            entityManager.getTransaction().commit();
        }catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }
    }
}
```

```sql
select * from product;
 id |  name
----+---------
  1 | product
  2 | Asus

select * from computer;
 ram  | id
------+----
 10GB |  2
```

In this approach:

- We won't have null columns.
- But we have to perform join operation. This make your queries slower.

## MappedSuperclass

The JPA standard specification defines the `@MappedSuperclass` annotation to allow an entity to inherit properties from a base class.

Unlike the `@Inheritance` annotation which maps the Java Object inheritance to a relational database model. `@MappedSuperclass` only models inheritance in the OOP world.

For instance, each entity we created must have the id field (with the `@Id` annotation), using `@MappedSuperclass` we can only create BaseEntity (which includes id field) once, then other entities can have the id field by extending the BaseEntity.

Let's say we have a base entity class `Vehicle` and two other classes `Car` and `Bicycle` which extends the `Vehicle`

Run the following sql queries:

```sql
CREATE TABLE car
(
    id SERIAL PRIMARY KEY, 	-- from mappedsuperclass
    color VARCHAR(100), 	-- from mappedsuperclass
    name VARCHAR(100)
);

CREATE TABLE bicycle
(
    id SERIAL PRIMARY KEY, -- from mappedsuperclass
    color VARCHAR(100),    -- from mappedsuperclass
    model VARCHAR(100)
);
```

Vehicle entity:

```java
@MappedSuperclass
public class Vehicle {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private String color;
}
```

Bicycle entity:

```java
@Entity
@Table(name = "bicycle")
public class Bicycle extends Vehicle {
    private String model;
}
```

Car entity:

```java
@Entity
@Table(name = "car")
public class Car extends Vehicle {
    private String name;
}
```

Now run the main method and look at the tables:

```java
public class Main {
   public static void main(String[] args) {
       Car car = new Car();
       car.setColor("black");
       car.setName("mercedes");

       Bicycle bicycle = new Bicycle();
       bicycle.setColor("brown");
       bicycle.setModel("m1");

       try {
           entityManager.getTransaction().begin();
           entityManager.persist(car);
           entityManager.persist(bicycle);
           entityManager.getTransaction().commit();
       }catch (Exception e) {
           e.printStackTrace();
       } finally {
           entityManager.close();
       }
   }
}
```

```sql
select * from car;
 id | color |   name
----+-------+----------
  1 | black | mercedes

select * from bicycle;
 id | color | model
----+-------+-------
  1 | brown | m1
```
