---
layout: post
title: "JPA Fundamentals & Hibernate - 9) AssociationOverride"
date: 2021-08-14 17:45:31 +0530
categories: "jpa"
author: "mehmetozanguven"
---

In this article, we are going to learn **how to create relationship using `@Embeddable` & `@Embedded` object** .

**`@AssociativeOverride` is used to override mappings of multiple relationship properties. To override a mapping defined by a mapped superclass or a relationship mapping from an embeddable class, `@AssociationOverride` is applied to the entity class.**

Topics are:

- [**Github Link**](#github_link)
- [**AssociationOverride**](#AssociationOverride)

## Github Link <a name="github_link"></a>

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/jpa_fundamentals_and_hibernate/tree/master/association-override)

First let's run the following sql queries (we have two tables called `department` and `employee`,
department can have many employees but employee can only belong to the one department):

```sql
CREATE TABLE employee
(
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department_id INT
);

CREATE TABLE department
(
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);
```

As you can see, we have simple one-to-many relationship. But this time, we will create a relationship over the `@Embeddable` & `@Embedded` annotations.

> Because Employee has the foreign key, it is the owner of the relationship

- Create `deparment` entity:

```java
@Entity
@Table(name = "department")
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;
    private String name;
    // getters and setters..
}
```

- Create `deparmentDetails` embeddable object (this object will hold the relationship)

> We will create the one-to-many relationship between employee and deparmant through the `department_details`

```java
@Embeddable
public class DepartmentDetails {
    @ManyToOne
    private Department department;
	// getters and setters
}
```

- Create `employee` entity:

```java
@Entity
@Table(name = "employee")
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;
    private String name;
    @Embedded
    private DepartmentDetails departmentDetails;
    // getters and setters
}
```

But the problem is how to create an relationship over the `deparment_id` using embeddable objects. To resolve the problem we should use `@AssociationOverride` annotation on the embedded object.

> If you only need to override the simple column name (column has no responsibility of the relationship), then you must use `@AttributeOverride` annotation.
>
> In our example, `department_id` is the one which creates the relationship between entities.

## AssociationOverride <a name="AssociationOverride"></a>

- It is used to override a mapping for an entity relationship.

```java
@AssociationOverride(name = "nameOfTheRelationshiProperty", joinColumns = @JoinColumn(name = "columnName"))
```

Now let's update the employee table:

```java
@Entity
@Table(name = "employee")
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private String name;

    @Embedded
    @AssociationOverride(name = "department", joinColumns = @JoinColumn(name = "department_id"))
    private DepartmentDetails departmentDetails;
}
```

After all we can run the main method:

```java
public class Main {
    public static void main(String[] args) {

        Department department = new Department();
        department.setName("d1");

        DepartmentDetails departmentDetails = new DepartmentDetails();
        departmentDetails.setDepartment(department);

        Employee employee = new Employee();
        employee.setName("e1");
        employee.setDepartmentDetails(departmentDetails);

        try {
            entityManager.getTransaction().begin();
            entityManager.persist(department);
            entityManager.persist(employee);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }
    }
}
```

If we look at the tables into database:

```sql
select * from department;
 id | name
----+------
  1 | d1
(1 row)

select * from employee;
 id | name | department_id
----+------+---------------
  1 | e1   |             1
```
