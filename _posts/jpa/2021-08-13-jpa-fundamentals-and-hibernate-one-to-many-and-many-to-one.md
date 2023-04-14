---
layout: post
title: "JPA Fundamentals & Hibernate - 7) One To Many & Many To One Relationship"
date: 2021-08-13 19:45:31 +0530
categories: "jpa"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/jpa/jpa-fundamentals-and-hibernate-one-to-many-and-many-to-one/"
---

> In the OneToMany setup, `Many` sides must have the foreign key and it is the owner of the relatonship

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

## Github Link

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/jpa_fundamentals_and_hibernate/tree/master/one-to-many-and-many-to-one)

## SQL Setup

We will have `Employee` and `Department` tables and Department can have many employee but employee can belong to the one department.

## Create one-to-many relationship without foreign key

In the one-to-many relationship, many side will have the foreign key. But before implementing with foreign key, let's look at the what will happen when there is no foreign key:

Run the following sql queries:

```sql
CREATE TABLE employee
(
   id SERIAL PRIMARY KEY,
   name VARCHAR(100)
);

CREATE TABLE department
(
   id SERIAL PRIMARY KEY,
   name VARCHAR(100)
);
```

### Create entities

- Department:

```java
@Entity
@Table(name = "department")
public class Deparment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private String name;

    // One Departmant to many employees
    @OneToMany
    private List<Employee> employees;
    // ...
}
```

- Employee:

```java
@Entity
@Table(name = "employee")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;
    private String name;
    // ...
}
```

### Run the main method

Create department and employee entities and persist them

```java
public class Main {
    public static void main(String[] args) {
        Employee employee = new Employee();
        employee.setName("Test");

        Deparment deparment = new Deparment();
        deparment.setName("Department");
        deparment.setEmployees(new ArrayList<>());
        deparment.getEmployees().add(employee);

        try {
            entityManager.getTransaction().begin();
            entityManager.persist(employee);
            entityManager.persist(deparment);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            System.out.println("Exception: " + e.getMessage());
        } finally {
            entityManager.close();
        }
    }
}
```

If you run the main method, you will get an exception:

```wiki
Caused by: org.postgresql.util.PSQLException: ERROR: relation "department_employee" does not exist
```

Here is the sql queries that run by the Hibernate:

```wiki
Hibernate:
    insert
    into
        employee
        (name)
    values
        (?)
Hibernate:
    insert
    into
        department
        (name)
    values
        (?)
Hibernate:
    insert
    into
        department_employee
        (Deparment_id, employees_id)
    values
        (?, ?)
```

As you can see, we have additional table called **department_employee**. If you don't specify the foreign key in the one-to-many relationship, JPA automatically create a new table to hold the corresponding department and employee ids.

We got an exception, because we don't want Hibernate update to our database. If you remove the comment related to the hbm2ddl in `persistence.xml`, and re-run the project, hibernate will create a new table.

```xml
        <properties>
           	uncomment this line
<!--            <property name="hibernate.hbm2ddl.auto" value="update" /> &lt;!&ndash; create / create-drop / update &ndash;&gt;-->
           ...
       </properties>
```

## Best way to model a one-to-many relationship

In this case, we will have two tables called `person & document` and person can have many documents but one document belongs to the only one person.

Please run the following sql queries:

```sql
CREATE TABLE person
(
    id SERIAL PRIMARY KEY,
    name VARCHAR(200)
);
```

```sql
CREATE TABLE document
(
    id SERIAL PRIMARY KEY,
    name VARCHAR(200),
    person_id INT
);
```

**The best way to model one-to-many relationship is to use just `@ManyToOne` annotation on the child-side (it is `document` in our example)**

First let's create the entities:

- Person entity:

```java
@Entity
@Table(name = "person")
public class Person {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;
    private String name;
    // getters and setters
}
```

- Document

```java

@Entity
@Table(name = "document")
public class Document {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "person_id")
    private Person person;

    // getters and setters
}

```

First let's persist one person only:

```java
public class Main {
    public static void main(String[] args) {
        Person person = new Person();
        person.setName("Ozan");
        try {
            entityManager.getTransaction().begin();
            entityManager.persist(person);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }
    }
}
```

Here is the sql result:

```sql
 select * from person;
 id | name
----+------
  1 | Ozan
(1 row)
```

Now if we want to add document to the **person Ozan**, we should find the person then add the document:

```java
public class Main {
    public static void main(String[] args) {
        try {
            entityManager.getTransaction().begin();
            Person ozan = entityManager.find(Person.class, 1L);
            Document document = new Document();
            document.setName("test document");
            document.setPerson(ozan);
            entityManager.persist(document);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }
    }
}
```

Here are the tables after run the main method:

```sql
testdatabase=# select * from person;
 id | name
----+------
  1 | Ozan
(1 row)

testdatabase=# select * from document;
 id |     name      | person_id
----+---------------+-----------
  1 | test document |         1
```

If we want to find all documents relaton the person Ozan, then we can use the JPQL(The Java Persistence Query Language), after getting the person id.

> I will create an article for the JPQL

## Bi-Directional one-to-many mapping

If possible, please use the best way approach.

> You won't find the bi-directional example in the github repo.

The idea with bidirectional one-to-many association is to allow you to keep a collection of child entities in the parent, and enable you to persist and retrieve the child entities via the parent entity.

- Update the Person class

```java
@Entity
@Table(name = "person")
public class Person {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private String name;

    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.LAZY, mappedBy = "person")
    private Set<Document> documents = new HashSet<>();
    // getters and setters
}
```

- Document entity will remain the same:

```java
public class Document {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "person_id")
    private Person person;
     // getters and setters
}
```

- Then you can save the person:

```java
Person person = new Person();
person.setName("Ozan");

// Create a Person
Person person = new Person();
person.setName("test");
// Create Documents
Document doc1 = new Document();
doc1.setName("doc1");
Document doc2 = new Document();
doc2.setName("doc2");

// add documents in the Person
person.getDocuments.add(doc1);
person.getDocuments.add(doc2);

// persist it
entityManager.persist(person);


// Retrieve Post
Person person = entityManager.find(Person.class, 1L);

// Get all the comments
Set<Documents> docs = person.getDocuments()
```

But this approach has some problems

### Problems with Bi-Directional Mapping

- Because we have no choose load documents without Person entity, we can not be able to limit the number of documents loaded. Therefore we can not be able to paginate.
- Because we are loading documents via Person entity, we can not be able to sort them based on different properties.
- Probable, we will encounter the exception called `LazyInitializitonException`

Last but not least, wait for the next post ...
