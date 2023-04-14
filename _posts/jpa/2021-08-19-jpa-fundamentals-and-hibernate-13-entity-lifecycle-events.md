---
layout: post
title: "JPA Fundamentals & Hibernate - 13) Entity Lifecycle Events (Final)"
date: 2021-08-19 22:45:31 +0530
categories: "jpa"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/jpa/jpa-fundamentals-and-hibernate-13-entity-lifecycle-events/"
---

In this article, we are going to learn lifecycle events in the JPA. As of the most application we will have `date_created` and `last_modified` column to our tables and we will update them via entity lifecycle listeners.

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

Before everything, let's create the table first:

```sql
CREATE TABLE product
(
    id SERIAL PRIMARY KEY,
    date_created TIMESTAMP,
    last_modified TIMESTAMP,
    name VARCHAR(100)
);
```

## Define the Product entity

If you read my previous articles, this step will be easy for you:

```java
@Entity
@Table(name = "product")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;
    @Column(name = "date_created")
    private LocalDateTime dateCreated;
    @Column(name = "last_modified")
    private LocalDateTime lastModified;
    private String name;
    // getters and setters
}
```

## JPA Entity Lifecycle Event:

When working with JPA, there are several events that can be notified of during an entity's lifecycle:

- before persist is called for a new entity – `@PrePersist`
- after persist is called for a new entity – `@PostPersist`
- before an entity is removed – `@PreRemove`
- after an entity has been deleted – `@PostRemove`
- before the update operation – `@PreUpdate`
- after an entity is updated – `@PostUpdate`
- after an entity has been loaded – `@PostLoad`

We have two options to use lifecycle event annotations:

1. Annotate the methods in the entity
2. Create entityListener class (abstract or interface) with annotated callback methods.

## Annotate the methods in the entity

Let's use the event operations in the entity:

```java
public class Product {
   	// ...
    public void setLastModified(LocalDateTime lastModified) {
        this.lastModified = lastModified;
    }

    @PrePersist
    public void prePersist() {
        System.out.println("Pre persist event called");
        this.dateCreated = LocalDateTime.now();
        this.lastModified = LocalDateTime.now();
    }

    @PreUpdate
    public void preUpdate() {
        System.out.println("Pre update event called");
        this.lastModified = LocalDateTime.now();
    }

    @PostLoad
    public void postLoad() {
        System.out.println("Post load event called");
        System.out.println("Entity: " + this + " was loaded");
    }

    @PreRemove
    public void preRemove() {
        System.out.println("PreRemove event called");
        System.out.println("Entity: " + this + " will be removed");
    }

    @PostRemove
    public void postRemove() {
        System.out.println("PostRemove event called");
        System.out.println("Entity: " + this + " was removed");
    }
    // ...
}
```

### PrePersist

Right now let's persist a product entity:

```java
public class Main {
    public static void main(String[] args) {
        try {
            entityManager.getTransaction().begin();
            Product product1 = new Product();
            product1.setName("product1");
            entityManager.persist(product1);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            System.out.println("Exception: " + e.getMessage());
        } finally {
            entityManager.close();
        }
    }
}
```

After run the main method, look at the console:

```wiki
Pre persist event called
Hibernate:
    insert
    into
        product
        (date_created, last_modified, name)
    values
        (?, ?, ?)
```

and also table:

```sql
select * from product;
 id |        date_created        |       last_modified        |   name
----+----------------------------+----------------------------+----------
  1 | 2021-08-12 12:32:36.414129 | 2021-08-12 12:32:36.414166 | product1
```

### PostLoad & PreUpdate

If we want to also see the how `PreUpdate` works:

```java
public class Main {
    public static void main(String[] args) {
        try {
            entityManager.getTransaction().begin();
            Product product1 = entityManager.find(Product.class, 1L);
            product1.setName("product1_v2");
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            System.out.println("Exception: " + e.getMessage());
        } finally {
            entityManager.close();
        }
    }
}
```

Console output:

```wiki
Hibernate:
    select
        ...

Post load event called
Entity: Product{id=1, dateCreated=2021-08-12T12:32:36.414129, lastModified=2021-08-12T12:32:36.414166, name='product1'} was loaded

Pre update event called
Hibernate:
    update
        ...

Process finished with exit code 0
```

and also table

```sql
select * from product;
 id |        date_created        |       last_modified        |    name
----+----------------------------+----------------------------+-------------
  1 | 2021-08-12 12:32:36.414129 | 2021-08-12 12:52:12.042188 | product1_v2
```

### PostLoad, PreRemove & PostRemove

Update the main class:

```java
public class Main {
    public static void main(String[] args) {
        try {
            entityManager.getTransaction().begin();
            Product product1 = entityManager.find(Product.class, 1L);
            entityManager.remove(product1);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            System.out.println("Exception: " + e.getMessage());
        } finally {
            entityManager.close();
        }
    }
}
```

Console output:

```wiki
Hibernate:
    select
        ...

---------------- First load the Product ----------------
Post load event called
Entity: Product{id=1, dateCreated=2021-08-12T12:32:36.414129, lastModified=2021-08-12T12:52:12.042188, name='product1_v2'} was loaded

---------------- PreRemove (indicates Product entity will be removed) ----------------
PreRemove event called
Entity: Product{id=1, dateCreated=2021-08-12T12:32:36.414129, lastModified=2021-08-12T12:52:12.042188, name='product1_v2'} will be removed

Hibernate:
    delete
    ...

---------------- PostRemove (indicates Product removed from the context and db) ----------------
PostRemove event called
Entity: Product{id=1, dateCreated=2021-08-12T12:32:36.414129, lastModified=2021-08-12T12:52:12.042188, name='product1_v2'} was removed

Process finished with exit code 0
```

## Create EntityListener class

We can abstract all the date operation into the one class called `GeneralEntity`

### Create abstract class (@MappedSuperclass)

```java
@MappedSuperclass
public abstract class GeneralEntity {

    @Column(name = "date_created")
    protected LocalDateTime dateCreated;
    @Column(name = "last_modified")
    protected LocalDateTime lastModified;

    // getters and setters

    @PrePersist
    public void prePersist() {
        System.out.println("Pre persist event called");
        this.dateCreated = LocalDateTime.now();
        this.lastModified = LocalDateTime.now();
    }

    @PreUpdate
    public void preUpdate() {
        System.out.println("Pre update event called");
        this.lastModified = LocalDateTime.now();
    }
}
```

Update the product entity:

```java
@Entity
@Table(name = "product")
public class Product extends GeneralEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;
    private String name;

    // getters and setters

    @PostLoad
    public void postLoad() {
        System.out.println("Post load event called");
        System.out.println("Entity: " + this + " was loaded");
    }

    @PreRemove
    public void preRemove() {
        System.out.println("PreRemove event called");
        System.out.println("Entity: " + this + " will be removed");
    }

    @PostRemove
    public void postRemove() {
        System.out.println("PostRemove event called");
        System.out.println("Entity: " + this + " was removed");
    }
}
```

Just try to insert another product and look at the table:

```sql
 select * from product;
 id |        date_created        |       last_modified        |   name
----+----------------------------+----------------------------+----------
  3 | 2021-08-12 13:08:04.639482 | 2021-08-12 13:08:04.639522 | product2

```

That's the final article for the mini JPA series.
