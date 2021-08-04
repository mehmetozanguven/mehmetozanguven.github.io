---
layout: post
title: "JPA Fundamentals & Hibernate - 1) Entities and Context"
date: 2021-08-04 12:45:31 +0530
categories: "jpa"
author: "mehmetozanguven"
---

In this series, we are going to learn the JPA fundamentals and we are going to use one of its implementation Hibernate.

Before diving into the series, make sure that:

- **You know the what is the maven**
- **You have experience with PostgreSQL (MySQL and others are also okey)**

Topics are:

- [**Github Link**](#github_link)
- [**Create Project**](#create_project)
- [**Persistence.xml**](#persistence_xml)
- [**PostgreSQL Configuration**](#postgresql_configuration)
- [**Entity**](#entity)
  - [**Create Entity class Product**](#product_entity)
- [**What do we mean by JPA Context?**](#what_jpa_context_is)
- [**Store Product Entity into Table**](#entity_to_table)
  - [**Persist is not an insert **](#persist_not_insert)
- [**Run the project**](#run_the_project)

## Github Link <a name="github_link"></a>

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/jpa_fundamentals_and_hibernate/tree/master/entity-and-context)

## Create Project <a name="create_project"></a>

First create new java project with the following dependencies:

```xml
	<properties>
        <hibernate.core.version>5.5.5.Final</hibernate.core.version>
        <postgresql.connector.version>42.2.23</postgresql.connector.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.hibernate</groupId>
            <artifactId>hibernate-core</artifactId>
            <version>${hibernate.core.version}</version>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>${postgresql.connector.version}</version>
        </dependency>
    </dependencies>
```

After that you should create a xml(usually called `persistence.xml`) file to write JPA configuration. (Under the `resources>META-INF` folder):

> If you are familiar with Spring Framework, you may notice that there is no xml file for JPA setup. Actually Spring Data provides one level of abstraction over the JPA. At the end, when you use Spring Framework, you only need to specify the database address in your properties files.

## Persistence.xml <a name="persistence_xml"></a>

With the persistence.xml file we can do the following things:

- the name of each persistence unit
- which managed persistence classes are part of a persistence unit,
- how these classes shall be mapped to database tables,
- the persistence provider that shall be used at runtime,
- ...

Here is the xml file for the postgresql: (you can search for the mysql on the internet)

```xml
<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence
             http://xmlns.jcp.org/xml/ns/persistence/persistence_2_2.xsd"
             version="2.2">
    <persistence-unit name="postgresqlPersistenceUnit" transaction-type="RESOURCE_LOCAL">

        <properties>
            <property name="javax.persistence.jdbc.driver" value="org.postgresql.Driver" /> <!-- DB Driver -->
            <property name="javax.persistence.jdbc.url" value="jdbc:postgresql://localhost:5432/testdatabase" /> <!-- DB Mane -->
            <property name="javax.persistence.jdbc.user" value="postgres" /> <!-- DB User -->
            <property name="javax.persistence.jdbc.password" value="1234" /> <!-- DB Password -->
	    <property name="hibernate.dialect" value="org.hibernate.dialect.PostgreSQL95Dialect"/> <!-- DB Dialect -->
            <property name="hibernate.hbm2ddl.auto" value="update" /> <!-- create / create-drop / update -->
            <property name="hibernate.show_sql" value="true" /> <!-- Show SQL in console -->
            <property name="hibernate.format_sql" value="true" /> <!-- Show SQL formatted -->
        </properties>

    </persistence-unit>

</persistence>
```

- `persistence-unit` is configuration of a connection to a specific database. Generally you will have only one persistence-unit
- `transaction-type` depends on the environment we deploy our application. If we use Jakarta EE environment, JPA expects that the container provides JTA-compliant connection provider. In Java SE environment, it uses a "RESOURCE_LOCAL" transaction.

> By default JPA persistence-unit includes all annotated managed classes found in its root. If we want to add classes that are located somewhere else, we can reference them in the persistence.xml file.

## PostgreSQL Configuration <a name="postgresql_configuration"></a>

Because we will use the local postgresql, we should create a sql tables in the postgresql

```sql
CREATE TABLE product
(
	id INT PRIMARY KEY,
    name VARCHAR(100),
    price NUMERIC,
    expiration_date DATE
);
```

> We didn't enable the auto increment for the id field. Basically we are saying that "As a programmer I will manage the id field, hey postgresql, do not manage id for me"

```bash
$ sudo -iu postgres
[postgres@localhost ~]$ psql
psql (12.7)
Type "help" for help.
postgres=# \c testdatabase
You are now connected to database "testdatabase" as user "postgres".
testdatabase=# CREATE TABLE product
testdatabase-# (
testdatabase(# id INT PRIMARY KEY,
testdatabase(#     name VARCHAR(100),
testdatabase(#     price NUMERIC,
testdatabase(#     expiration_date DATE
testdatabase(# );
CREATE TABLE

testdatabase=# select * from product ;
 id | name | price | expiration_date
----+------+-------+-----------------
(0 rows)

```

## Entity <a name="entity"></a>

In JPA, we will map the Java Class(es) with the table(s) is called **entity**.

Or in other words, entities are the objects what we store in the database.

### Create Entity class Product <a name="product_entity"></a>

```java
package entities;
import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.Id;
import java.time.LocalDate;

@Entity
@Table(name = "product") // sql table name
public class Product {

    @Id // every entity should have an ID
    private int id;

    private String name;
    private double price;

    @Column(name = "expiration_date")
    private LocalDate expirationDate;

    // getters and setters...
}
```

- To make this class as entity, just use the `@Entity` annotation.
- `@Table` is used to indicate that "this entity is map to the table called product in the database"
- Any entity should have or at least inherit an ID, ID is the primary key.
- Every columns in table should be mapped with the fields of the class.

> We have used @Column annotation to indicate that "map the field `expirationDate` with the column name `expiration_date` in the table"
>
> If the both column name and the field name is the same, you don't need to specify the @Column annotation

## What do we mean by JPA Context? <a name="what_jpa_context_is"></a>

Entity Manager is responsible to manage entities. What it manages is the context of the entities.

JPA implementation is creating own its context. Context is actually collection of instances. Those instances are some type of managed objects. For the JPA, we call it "entities".

## Store Product Entity into Table <a name="entity_to_table"></a>

To save, update or delete our entities, we should need an instance of entity manager.

To obtain entity manager, we must create an instance of entity manager factory. Factory will create the entity manager.

```java
public class Main {
    public static final String PERSISTENCE_UNIT = "postgresqlPersistenceUnit";

    public static void main(String[] args) {
        EntityManagerFactory entityManagerFactory = Persistence.createEntityManagerFactory(PERSISTENCE_UNIT);
        EntityManager entityManager = entityManagerFactory.createEntityManager();

        Product product = new Product();
        product.setId(1);
        product.setName("Cheese");
        product.setPrice(5.4);
        product.setExpirationDate(LocalDate.now());

        try {
            entityManager.getTransaction().begin();
            entityManager.persist(product);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            System.out.println("Exception " + e.getMessage());
        }
    }
}
```

- We must set id, because in the beginning, we determined that we are responsible for assigning ids.
- Every "change operation" in the JPA, should be represented in the transactions. You should get the current transaction `entityManager.getTransaction()` and then begin transaction `...getTransaction().begin()`. After all you should commit the transaction `....getTransaction().commit()` . In the case of failure you should catch the exception and the rollback.

### Persist is not an insert <a name="persist_not_insert"></a>

- `entityManager.persist(product);` **is used to add the instance in the context.**
- Calling persist method is not directly an insert to the database.
- When the transaction is finished by the calling of `entityManager.getTransaction().commit()`, the context (managed by the entity manager) will be mirrored to the database.

## Run the project <a name="run_the_project"></a>

After you run the main method, look at the console and the table

```wiki
INFO: HHH000490: Using JtaPlatform implementation: [org.hibernate.engine.transaction.jta.platform.internal.NoJtaPlatform]
Hibernate:
    insert
    into
        product
        (expiration_date, name, price, id)
    values
        (?, ?, ?, ?)
```

```sql
select * from product ;
 id |  name  | price | expiration_date
----+--------+-------+-----------------
  1 | Cheese |   5.4 | 2021-08-01
```

As you can see, we have persisted our first entity into the database.

Last but not least, wait for the next post...
