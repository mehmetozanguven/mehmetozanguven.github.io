---
layout: post
title: "JPA Fundamentals & Hibernate - 3) @Enumareted, @Temporal and Date Types"
date: 2021-08-06 12:45:31 +0530
categories: "jpa"
author: "mehmetozanguven"
---

In this post, we are going to learn **`@Enumarated` and `@Temporal`** annotations, **how to store enums as ordinal and string** and **when to store enum types to the database.** Also we are going to learn **how to work with date types and when we need to use @Temporal annotation**

Topics are:

- [**Github Link**](#github_link)
- [**SQL Queries**](#sql_queries)
- [**Currency Enum Class**](#curreny_enum)
- [**Create Price entity class**](#price_entity_class)
- [**Store enum as ORDINAL**](#enum_as_ordinal)
- [**Store enum as String**](#enum_as_string)
- [**Which EnumType should we use(Ordinal vs String)**](#which_enum_type)
- [**Working with date**](#work_with_date)
- [**Working with @Temporal**](#work_with_temporal)


## Github Link <a name="github_link"></a>

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/jpa_fundamentals_and_hibernate/tree/master/enumarated-and-temporal)

## SQL Queries <a name="sql_queries"></a>

Create table called price, which has a currency column. Enum values will be used for currency column.

```sql
CREATE TABLE price
(
	id SERIAL PRIMARY KEY,
    amount NUMERIC,
    currency INT
);
```

## Currency Enum Class <a name="curreny_enum"></a>

Pretty-simple :

```java

public enum Currency {
    DOLLAR, EURO, TL
}
```

## Create entity class <a name="price_entity_class"></a>

```java
@Entity
@Table(name = "price")
public class Price {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private double amount;

    @Enumerated(EnumType.ORDINAL) // ORDINAL is the default value
    private Currency currency;
}
```

## Store enum as ORDINAL <a name="enum_as_ordinal"></a>

When you use EnumType.ORDINAL:

- `Currency.DOLLAR` will be stored as 0 (zero)
- `Currency.EURO` : will be stored as 1
- `Currency.TL` : will be stored as 2.

Let's run the main method:

```java
public class Main {
    public static final String PERSISTENCE_UNIT = "postgresqlPersistenceUnit";

    public static void main(String[] args) {
 		// ...
        Price price = new Price();
        price.setAmount(10.34);
        price.setCurrency(Currency.TL);

        Price anotherPrice = new Price();
        anotherPrice.setAmount(5.34);
        anotherPrice.setCurrency(Currency.EURO);

        try {
            entityManager.getTransaction().begin();
            entityManager.persist(price);
            entityManager.persist(anotherPrice);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            System.out.println("Exception: " + e.getMessage());
        } finally {
            entityManager.close();
        }
    }
}
```

After that look at the table in the database:

```sql
select * from price;
-- 2 = TL, 1 = DOLLAR
 id | amount | currency
----+--------+----------
  1 |  10.34 |        2
  2 |   5.34 |        1
(2 rows)

```

## Store enum as String <a name="enum_as_string"></a>

- Re-create the price table with the following sql:

```sql
DROP TABLE price ;
CREATE TABLE price
(
	id SERIAL PRIMARY KEY,
    amount NUMERIC,
    currency VARCHAR(100)
);
```

- Update the `@Enumarated` annotation:

```java
@Enumerated(EnumType.STRING)
private Currency currency;
```

- Run the main method again and look at the table in the database:

```sql
select * from price;
 id | amount | currency
----+--------+----------
  1 |  10.34 | TL
  2 |   5.34 | EURO
(2 rows)
```

## Which EnumType should we use? <a name="which_enum_type"></a>

Whenever it is possible, please use the `EnumType.STRING`. Let's say after period of time we need to add another currency :

```java
public enum Currency {
    DOLLAR, EURO, TL, GBP
}
```

If you add GBP at the end, you will not face any problem (when using ORDINAL). However if you place GBP at the beginning, index 0 will become GBP and index 3 will be become DOLLAR and it leads to the corruption in your database. Because most probably you will have records with index 0 which indicate DOLLAR.

## Working with Dates <a name="work_with_date"></a>

The PostgreSQL JDBC driver implements native support for the Java 8 Date and Time API (JSR-310) using JDBC 4.2.

| PostgreSQL                     | Java SE 8      |
| ------------------------------ | -------------- |
| DATE                           | LocalDate      |
| TIME [ WITHOUT TIMEZONE ]      | LocalTime      |
| TIMESTAMP [ WITHOUT TIMEZONE ] | LocalDateTime  |
| TIMESTAMP WITH TIMEZONE        | OffsetDateTime |

Let's learn how to use Date types in the JPA.

> JPA 2.2 supports `LocalDate`, so no converter is needed.
>
> Hibernate also supports it as of 5.3 version.
>
> You don't need to convert any of the following date:
>
> ```java
> java.time.LocalDate
> java.time.LocalTime
> java.time.LocalDateTime
> java.time.OffsetTime
> java.time.OffsetDateTime
> ```

First create a table called `simple_date`:

```sql
CREATE table simple_date
(
	id SERIAL PRIMARY KEY,
	only_date DATE,
    only_time TIME,
    date_with_time TIMESTAMP
);
```

Then create an entity class for the table:

```java
@Entity
@Table(name = "simple_date")
public class SimpleDate {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column(name = "only_date")
    private LocalDate onlyDate;

    @Column(name = "only_time")
    private LocalTime localTime;

    @Column(name = "date_with_time")
    private LocalDateTime dateWithTime;
}
```

Run the main method:

```java
public class Main {
    public static void main(String[] args) {
        SimpleDate date = new SimpleDate();
        date.setOnlyDate(LocalDate.now());
        date.setLocalTime(LocalTime.now());
        date.setDateWithTime(LocalDateTime.now());

        SimpleDate anotherDate = new SimpleDate();
        anotherDate.setOnlyDate(LocalDate.now());
        anotherDate.setLocalTime(LocalTime.now());
        anotherDate.setDateWithTime(LocalDateTime.now());
        try {
            entityManager.getTransaction().begin();
            entityManager.persist(date);
            entityManager.persist(anotherDate);
            entityManager.getTransaction().commit();
        }catch (Exception e) {
            System.out.println("Exception: " + e.getMessage());
        } finally {
            entityManager.close();
        }
    }
}
```

Look at the table:

```sql
select * from simple_date;
 id | only_date  | only_time |       date_with_time
----+------------+-----------+----------------------------
  1 | 2021-08-01 | 17:46:04  | 2021-08-01 17:46:04.219018
  2 | 2021-08-01 | 17:46:04  | 2021-08-01 17:46:04.219038
```

## Working with @Temporal <a name="work_with_temporal"></a>

To work with java `Date or Calendar` package, you need to use `@Temporal` annotation. Because Java doesn't really know how to directly relate to SQL types.

> `java.util Date and Calendar` are legacy packages. It is better to use `java.time.package.`

Here is the simple example:

```java
@Entity
@Table(name = "simple_date")
public class SimpleDate {
	// ...
    @Column(name = "date")
    @Temporal(TemporalType.DATE)
    private Date date;

    @Temporal(value=TemporalType.TIMESTAMP)
    @Column(name="timestampm")
    private Date updatedTime;
    // ...
}
```

Last but not least, wait for the next post ...
