---
layout: post
title: "JPA Fundamentals & Hibernate - 8) JPA Fundamentals & Hibernate - 8) Many To Many Relationship"
date: 2021-08-13 19:45:31 +0530
categories: "jpa"
author: "mehmetozanguven"
---

In this post, we are going to learn **how to create many-to-many relationship in JPA**.

Topics are:

- [**Github Link**](#github_link)
- [**One directional Relationship**](#one_directional)
- [**Bi-Directional Relationship**](#bi_drectional)

## Github Link <a name="github_link"></a>

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/jpa_fundamentals_and_hibernate/tree/master/many-to-many)

We will create two tables called `student & professor` . And in our database, one professor could have multiple students, and one student could have multiple professors. If tables have have many-to-many relationship, then we should always create a separate table where we store relation.

Run the following sql queries:

```sql
CREATE TABLE student
(
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE professor
(
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE professor_student
(
    professor_id INT,
    student_id INT
);
```

As the other relationship (one-to-one and one-to-many), we have two options:

- One directional relationship
- Bi-directional relationship

## One directional Relationship <a name="one_directional"></a>

In the many-to-many relationship, we don't have exact owner of the relation. Therefore we should specify one side as the owner which will include both `@ManyToMany` and `@JoinTable` annotations.

> `@JoinTable` will be always on the side of the owner regardless of the relationship type (one-to-one, one-to-many many-to-many etc..)

In our example, I choose the `Professor` as the owner of the relationship.

- Create `Professor` entity:

```java
@Entity
@Table(name = "professor")
public class Professor {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;
    private String name;
    @ManyToMany
    @JoinTable(name = "professor_student", // table name
            joinColumns = @JoinColumn(name = "professor_id"),
            inverseJoinColumns = @JoinColumn(name = "student_id")
    )
    private List<Student> students;
}
```

- Run the main method:

```java
public class Main {
    public static void main(String[] args) {
        Student s1 = new Student();
        s1.setName("s1");

        Professor p1 = new Professor();
        p1.setName("p1");
        p1.setStudents(new ArrayList<>());
        p1.getStudents().add(s1);

        try {
            entityManager.getTransaction().begin();
            entityManager.persist(p1);
            entityManager.persist(s1);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }
    }
}
```

Let's look at the sql tables:

```sql
select * from professor;
 id | name
----+------
  1 | p1

select * from student;
 id | name
----+------
  1 | s1

select * from professor_student ;
 professor_id | student_id
--------------+------------
            1 |          1
```

## Bi-Directional Relationship <a name="bi_drectional"></a>

After choosing the owner of the relationship, we will just use the `@ManyToMany(mappedBy="fieldNameOnTheOwnerSide")` annotation on the other side which will be the `Student` in our example.

- Update the `Student` class

```java
@Entity
@Table(name = "student")
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;
    private String name;
    @ManyToMany(mappedBy = "students")
    private List<Professor> professors;
    // getters and setters ...
}
```

- And when we create bi-directional relationship always both sides must know each other. Update the main class:

```java
public class Main {
    public static void main(String[] args) {
        Student s1 = new Student();
        s1.setName("s1");

        Professor p1 = new Professor();
        p1.setName("p1");
        p1.setStudents(new ArrayList<>());
        p1.getStudents().add(s1);

        s1.setProfessors(new ArrayList<>());
        s1.getProfessors().add(p1);

        try {
            entityManager.getTransaction().begin();
            entityManager.persist(p1);
            entityManager.persist(s1);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }
    }
}
```

That's it for the many to many relationship, wait it for the one ...
