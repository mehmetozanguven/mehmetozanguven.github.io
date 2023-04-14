---
layout: post
title: "JPA Fundamentals & Hibernate - 11) Entity Manager"
date: 2021-08-15 15:45:31 +0530
categories: "jpa"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/jpa/jpa-fundamentals-and-hibernate-entitiy-manager/"
---

In this article, we are going to discuss **what is the entity manager in the JPA** and also **learn the basic operations in the entity manager**

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

## How to Obtain EntityManager

To obtain an entityManager, you must use factory pattern. For all the previous examples, we always used the factory pattern to obtain an EntityManager:

```java
public class Main {
    public static final String PERSISTENCE_UNIT = "postgresqlPersistenceUnit";

    public static void main(String[] args) {
        EntityManagerFactory entityManagerFactory = Persistence.createEntityManagerFactory(PERSISTENCE_UNIT);
        EntityManager entityManager = entityManagerFactory.createEntityManager();
        // ...
    }
```

## What is the EntityManager

**Entity Manager is responsible to manage entities. What it manages is the context of the entities.**

**JPA implementation is creating own its context. Context is actually collection of instances. Those instances are some type of managed objects. For the JPA, we call it “entities”.**

## Basic Operations with EntityManager

I will not give examples for all the operations. And actually you won't need all the operations especially if use framework such as Spring Boot.

### persist() operation

Use this operation to make instance part of a context. In other words, to make instance to be managed by JPA.

### flush() operation

Synchronize the context to the underlying database.

---

### find() operation

To retrieve instance of an entity from the database by using the primary key

### getReference() operation

Similar to `find()` but this operation doesn't give all the details in the beginning.

### contains(Object entity) operation

Check if the instance is or not in the context

---

### detach(Object entity) operation

Remove the given entity from the context

### clear() operation

Clear the context. In other word, detach all the instances from the context.

---

### remove() operation

Remove the entity instance from the context and the persistence layer.

---

### merge(Object entity) operation

Merge the state of the given entity into the current context.

### refresh(Object entity) operation

Refresh the state of the instance from the database, overwriting changes made to the entity, if any.

## Difference between persist() and flush()

Use `flush()` if we want to reflect the context as soon as possible.

When we are using `persis(Object entity)`, instance will reflected to the entity after commit the transaction.

See the examples:

- If you run only persist then commit :

```java
public class Main {
    public static void main(String[] args) {
        try {
            entityManager.getTransaction().begin();
            entityManager.persist(...);
            System.out.println("This will be run before commit");
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }
    }
}
// console output:
// This will be run before commit
// Hibernate: insert into Cart (name, id) values (?, ?)
```

- If you run the flush method after the persist:

```java
public class Main {
    public static void main(String[] args) {
        try {
            entityManager.getTransaction().begin();
            entityManager.persist(...);
            entityManager.flush();
            System.out.println("This will be run before commit");
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }
    }
}
// console output:
// Hibernate: insert into Cart (name, id) values (?, ?)
// This will be run before commit
```

As we can see, flush will run the insert sql query before the commit method.

## Difference between find() and getReference()

`getReference()` is a kind of lazy loading version of the `find()` method.

When we get the entity with the `find()`, Hibernate will run the select query at the beginning. However when we use `getReference()` until we use returned object somewhere else (such as `returnedObject.getId()`) hibernate will not run the select query.

In other words, `getReference()` returns the proxy of the entity and until you use proxy, hibernate will not run any select query.

Last but not least, wait for the next one ...
