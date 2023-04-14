---
layout: post
title: "JPA Fundamentals & Hibernate - 5) Composed Keys"
date: 2021-08-10 19:45:31 +0530
categories: "jpa"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/jpa/jpa-fundamentals-and-hibernate-composed-primary-keys/"
---

In this article, we are going to find out **how to create composite keys** in JPA.

> Composite key is the key which composed of multiple columns.

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

## Github Link

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/jpa_fundamentals_and_hibernate/tree/master/composed-keys)

Let's say we have table called `departmant` and it has composed primary keys (for column code and number).

## SQL Setup

Run the following sql query:

```sql
CREATE TABLE department
(
    code VARCHAR(100),
    number INT,
    name VARCHAR(100),
    PRIMARY KEY(code, number)
);
```

## Create entities

We have options to create composite key:

### Create Composed Keys with `@IdClass`

- In the entity class, annotate all the primary keys field with `@Id`
- Create a class which represents the composed key
  - This class must have the same fields name as in the entity class
  - Also this class must implements `Serializable` interface (JPA specification)
- Annotate entity class with `@IdClass(YourComposedClass.class)`

#### Annotate all the primary keys field with `@Id`

```java
@Entity
@Table(name = "department")
public class Department {
    @Id
    private String code;
    @Id
    private int number;
    private String name;

    // getters and setters..
}
```

#### Create a Composed key class

Fields in the composed key class must match with the fields annotated with `@Id` in the entity class

```java
public class DepartmentPK implements Serializable {
    private String code;
    private int number;
    // getters and setters
}
```

#### Annotate entity class with `@IdClass`

```java
@IdClass(DepartmentPK.class)
public class Department {}
```

Let's create an department entity and persist it:

```java
public class Main {

    public static void main(String[] args) {
        Department department = new Department();
        department.setNumber(10);
        department.setCode("AB");
        department.setName("test");

        try {
            entityManager.getTransaction().begin();
            entityManager.persist(department);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            System.out.println("Exception: " + e.getMessage());
        } finally {
            entityManager.close();
        }
    }
}
```

Then run the select query:

```sql
select * from department ;
 code | number | name
------+--------+------
 AB   |     10 | test
(1 row)
```

### Create Composed Keys with Embeddable

Almost the same procedure as the `@Embeddable /@Embedded` in the previous blog. Only difference is that we should use `@EmbeddedId` and embeddable object must implements Serializable interface

- First, create a new table called `department_two`

```sql
CREATE TABLE department_two
(
    code VARCHAR(100),
    number INT,
    name VARCHAR(100),
    PRIMARY KEY(code, number)
);
```

- Next create embeddable composed key object:

```java
@Embeddable
public class DepartmentEmbeddablePK implements Serializable {
    private int number;
    private String code;
    // getters and setters
}
```

- Create an entity class:

```java
@Entity
@Table(name = "department_two")
public class DepartmentTwo {

    @EmbeddedId
    @AttributeOverrides({
            @AttributeOverride(name = "number", column = @Column(name = "number")),
            @AttributeOverride(name = "code", column = @Column(name = "code")),
    })
    private DepartmentEmbeddablePK departmentEmbeddablePK;
    private String name;
    // getters and setters
}
```

- Finally create an entity object and persist it:

```java
public class Main {
    public static void main(String[] args) {
        DepartmentEmbeddablePK departmentEmbeddablePK = new DepartmentEmbeddablePK();
        departmentEmbeddablePK.setNumber(11);
        departmentEmbeddablePK.setCode("BC");

        DepartmentTwo departmentTwo = new DepartmentTwo();
        departmentTwo.setName("test2");
        departmentTwo.setDepartmentEmbeddablePK(departmentEmbeddablePK);

        try {
            entityManager.getTransaction().begin();
            entityManager.persist(departmentTwo);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            System.out.println("Exception: " + e.getMessage());
        } finally {
            entityManager.close();
        }
    }
}
```
