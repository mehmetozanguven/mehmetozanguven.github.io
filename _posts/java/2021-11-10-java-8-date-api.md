---
layout: post
title: "Java 8 Date and Time API in one tutorial"
date: 2021-11-10 19:45:31 +0530
categories: "java"
author: "mehmetozanguven"
---

Even we have Java 17 version right now, industries still are using Java 8 & 11 even old ones using Java 6. Also many of these industries are using old legacy date representation called `Date` & `Calendar`. Java 8 provides new way to create dates with time api which is more convenient and also more easy to use.

> There is no reason to use legacy `Date` & `Calendar` APIs in the 2021.

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

First we should answer the "why java needs new api for dates"?

## Reason for new date api

We can group the reasons into the three categories:

- **Thread safety**: `Date` & ̀ Calender` aren't thread-safe. Developers should think twice when they are dealing with concurrency problems. The new API avoids this issue because new classes are immutable.
- **Difficult to understand**: Legacy APIs are hard to understand and their designs are quite poor. The new APIs clearly define different use cases for `Date`and `Time`
- **Zoned Date**: In the legacy APIs, to work with time-zone logics we must have to write additional code blocks. The new APIs handles zoned date more clear and easy way.

Most probably you are working on **Local** and **Zoned** classes in the `java.time.package`

## LocalDate & LocalTime & LocalDateTime

The `java.time.LocalDate` and the `java.time.LocalTime` classes provide a representation of date and time without timezones. These classes does not represent a time zone. They can be used for instance birthdays.

- **LocalDate**: represents ISO-8601 calendar system, such as 2021-12-04
- **LocalTime**: represents time without a date in the ISO-8601 calendar system, such as 22:12:45
- **LocalDateTime** is kind of a combination both **LocalDate & LocalTime**, such as 2021-11-24T23:12:30

**IMPORTANT NOTE: LocalDate, LocalTime and LocalDateTime have no time-zone. That doesn't mean that they are something from 1970 UTC. Local means that not zoned, no offset, no UTC etc..**

### Creating LocalDate

- Print current date:

```java
public static void main(String[] args) {
        LocalDate now = LocalDate.now();
        System.out.println(now);
    	// 2021-10-27
}
```

- We can also create local date from `of` and `parse` method:

```java
public class Test {

    public static void main(String[] args) {
        // public static LocalDate of(int year, int month, int dayOfMonth)
        LocalDate now = LocalDate.of(2021, 10, 27);
        System.out.println(now); // 2021-10-27

        LocalDate parsed = LocalDate.parse("2021-09-27");
        System.out.println(parsed); // 2021-09-27
    }
}
```

- We can add years, months & days to the local date using `ChronoUnit`:

```java
public class Test {

    public static void main(String[] args) {
        LocalDate today = LocalDate.now();
        System.out.println("Today: " + today); // Today: 2021-10-27

        LocalDate tomorrow = today.plus(1, ChronoUnit.DAYS);
        System.out.println("Tomorrow " + tomorrow); // Tomorrow 2021-10-28

        LocalDate twoMonthsLater = today.plus(2, ChronoUnit.MONTHS);
        System.out.println("Two months later: " + twoMonthsLater); // Two months later: 2021-12-27

        LocalDate oneYearLater = today.plus(1, ChronoUnit.YEARS);
        System.out.println("One year later: " + oneYearLater); // One year later: 2022-10-27
    }
}
```

> We can only add supported ChronoUnit, otherwise we will get an `UnsupportedTemporalTypeException`. Just inspect the plus method:
>
> ```java
>  @Override
>     public LocalDate plus(long amountToAdd, TemporalUnit unit) {
>         if (unit instanceof ChronoUnit) {
>             ChronoUnit f = (ChronoUnit) unit;
>             switch (f) {
>                 case DAYS: return plusDays(amountToAdd);
>                 case WEEKS: return plusWeeks(amountToAdd);
>                 case MONTHS: return plusMonths(amountToAdd);
>                 case YEARS: return plusYears(amountToAdd);
>                 case DECADES: return plusYears(Math.multiplyExact(amountToAdd, 10));
>                 case CENTURIES: return plusYears(Math.multiplyExact(amountToAdd, 100));
>                 case MILLENNIA: return plusYears(Math.multiplyExact(amountToAdd, 1000));
>                 case ERAS: return with(ERA, Math.addExact(getLong(ERA), amountToAdd));
>             }
>             throw new UnsupportedTemporalTypeException("Unsupported unit: " + unit);
>         }
>         return unit.addTo(this, amountToAdd);
>     }
> ```

- We can also get the day of the week, month and more:

```java
public class Test {

    public static void main(String[] args) {
        LocalDate today = LocalDate.now();

        System.out.println("Today: " + today);
        System.out.println("Day of the week: " + today.getDayOfWeek());
        System.out.println("Month: " + today.getMonth());
        System.out.println("Day of the month: " + today.getDayOfMonth());
        // Today: 2021-10-27
		// Day of the week: WEDNESDAY
		// Month: OCTOBER
		// Day of the month: 27
    }
}
```

- Finally we can check whether date is before or after the specified date:

```java
public class Test {

    public static void main(String[] args) {
        boolean isBefore = LocalDate.parse("2021-10-26").isBefore(LocalDate.parse("2021-10-26"));
        System.out.println(isBefore); // false
    }
}
```

### Creating LocalTime

- Print current time:

```java
public class Test {

    public static void main(String[] args) {
        LocalTime now = LocalTime.now();
        System.out.println("Now: " + now);
        // Now: 22:55:18.517967
    }
}
```

- We can also create local time from `of` and `parse` method:

```java
public class Test {

    public static void main(String[] args) {
        LocalTime of = LocalTime.of(15, 45, 12);
        System.out.println("Time: " + of); // Time: 15:45:12

        LocalTime parsed = LocalTime.parse("03:12");
        System.out.println("Parsed: " + parsed); // Parsed: 03:12
    }
}
```

- As we did LocalDate, we can also add or minus time from LocalTime using supported ChronoUnit:

```java
public class Test {

    public static void main(String[] args) {
        LocalTime now = LocalTime.now();
        LocalTime twoHoursBefore = now.minus(2, ChronoUnit.HOURS);

        System.out.println("Now: " + now);
        System.out.println("Two hours before: " + twoHoursBefore);
        //        Now: 22:59:46.954954
		//        Two hours before: 20:59:46.954954
    }
}
```

## ZoneId vs ZonedOffset

Before diving into ZonedDateTime and OffsetDateTime, it is crucial to understand ZonedId and ZonedOffset.

**ZoneId is a representation of the time-zone such as Europe/Istanbul**

We have two practical implementation for Zone Id:

- First one is fixed offset from UTC (called ZoneOffset) (using with OffsetDateTime)
- Second one uses geographical region and applies set of rules, then set the offset from UTC (called ZoneId) (using with ZonedDateTime)

For instance when you use ZoneId for Europe/Berlin and because Berlin changes its time in the summer (turning to summer time or known as DST), your time in the java code will also change. ZoneId has aware of this situation.

But if you use ZoneOffset, there will be no change.

## ZonedDateTime

Represents date-time with a time-zone in the ISO-8601 calendar system such as 2021-10-26T10:23:30+01:00 Europe/Paris

- Print all available zone ids:

```java
public class Test {

    public static void main(String[] args) {
        ZoneId.getAvailableZoneIds().forEach(zone -> System.out.println(zone));

        /*
        ...
        Australia/Lindeman
		America/Los_Angeles
		SystemV/EST5EDT
		Pacific/Majuro
		America/Argentina/Buenos_Aires
		...
		*/
    }
}
```

- After creating `ZoneId`, we can convert the `LocalDateTime` to zoned date time:

```java
public class Test {

    public static void main(String[] args) {
        LocalDateTime now = LocalDateTime.now();
        ZoneId utc = ZoneId.of("UTC");
        ZoneId newYork = ZoneId.of("America/New_York");
        ZoneId istanbul = ZoneId.of("Europe/Istanbul");

        ZonedDateTime utcTime = ZonedDateTime.of(now, utc);
        ZonedDateTime newYorkTime = ZonedDateTime.of(now, newYork);
        ZonedDateTime istanbulTime = ZonedDateTime.of(now, istanbul);

        System.out.println("UTC time: " + utcTime);
        System.out.println("America/NewYork time: " + newYorkTime);
        System.out.println("Europe/Istanbul time: " + istanbulTime);

//        UTC time: 2021-10-27T23:14:40.689322Z[UTC]
//        America/NewYork time: 2021-10-27T23:14:40.689322-04:00[America/New_York]
//        Europe/Istanbul time: 2021-10-27T23:14:40.689322+03:00[Europe/Istanbul]
    }
}
```

- We can also use `ZonedDateTime.now()` or `onedDateTime now(ZoneId zone)` methods
- To get system default time-zone:

```java
public class Test {
    public static void main(String[] args) {
        System.out.println(ZoneId.systemDefault());
        // Europe/Istanbul
    }
}
```

## OffsetDateTime

**OffsetDateTime** class represent a date-time with an offset from UTC/Greenwich in the ISO-8601 calendar system.

We must use this class when working with database.

> **OffsetDateTime**, **ZonedDateTime** and **Instant** all store an instant on the time-line to nanosecond precision. Instant is the simplest, simply representing the instant. **OffsetDateTime** adds to the instant the offset from UTC/Greenwich, which allows the local date-time to be obtained. ZonedDateTime adds full time-zone rules

If dates applies set of specific rules according to geographical regions, then it prints with **2021-10-27T23:14:40.689322Z[UTC]** or **2021-10-27T23:14:40.689322+03:00[Europe/Istanbul]**. Here is the example:

```java
public class Test {

    public static void main(String[] args) {
        ZoneId istanbul = ZoneId.of("Europe/Istanbul");
        LocalDateTime now = LocalDateTime.now();
        System.out.println("LocalDateTime now: " + now);
//        LocalDateTime now: 2021-10-28T01:45:06.737242

        ZonedDateTime DSTAwareIstanbulTime = ZonedDateTime.of(now, istanbul);
        System.out.println("Istanbul time zone with DST rules: " + DSTAwareIstanbulTime);
//        Istanbul time zone with DST rules: 2021-10-28T01:45:06.737242+03:00[Europe/Istanbul]

        ZoneOffset istanbulZoneOffset = DSTAwareIstanbulTime.getOffset();
        System.out.println("ZoneOffset to setup fixed time zone: " + istanbulZoneOffset);
//        ZoneOffset to setup fixed time zone: +03:00

        OffsetDateTime fixedOffsetIstanbulTime =
                OffsetDateTime.of(now, DSTAwareIstanbulTime.getOffset());
        System.out.println("Istanbul time zone without DST rules: " + fixedOffsetIstanbulTime);
//        Istanbul time zone without DST rules: 2021-10-28T01:45:06.737242+03:00
    }
}
```

- Examples for `OffsetDateTime.now`:

```java
public class Test {

    public static void main(String[] args) {
        OffsetDateTime utcNow = OffsetDateTime.now(ZoneId.of("UTC"));
        OffsetDateTime europeIstanbul = OffsetDateTime.now(ZoneId.of("Europe/Istanbul"));

        System.out.println("UTC Now: " + utcNow);
        System.out.println("Europe/Istanbul: " + europeIstanbul);
//        UTC Now: 2021-10-27T22:13:13.278057Z
//        Europe/Istanbul: 2021-10-28T01:13:13.291924+03:00
    }
}
```

## Difference Between ZonedDateTime and OffsetDateTime

**The difference between OffsetDateTime and ZonedDateTime is that: in ZonedDateTime, ZoneId is responsible for changing the offset, we can't have fully control on it. This means that ZonedId is aware of DST (daylight save time)**

> **Daylight saving time** (**DST**) (or called as **summer time**), is the practice of advancing clocks (typically by one hour) during warmer months so that darkness falls at a later clock time. For more information here is the wiki [link](https://en.wikipedia.org/wiki/Daylight_saving_time)

For instance, if the difference from UTC and your timezone is (let's say) -05:00, after six months this difference could be -04:00 (according to the DST rules), when you use ZonedDateTime.

If you use OffsetDateTime, the difference is the one you are set when you construct the offset. That's is the reason why we should use OffsetDateTime in the database. (-05:00 to -04:00 could lead unexpected income failures or bugs in the application)

## Periods

Period represents a distance on the timeline, such as 5 months and 1 day. After that we can add or minus amount of period from dates.

**Period distance is based on ISO-8601 calendar system, such as 1 year, 5 months and 1 day**

```java
public class Test {

    public static void main(String[] args) {
        Period fiveMonthAndOneDayPeriod = Period.of(0, 5, 1);
        LocalDate now = LocalDate.now();

        System.out.println("Now: " + now);
        System.out.println("After five months and one day: " + now.plus(fiveMonthAndOneDayPeriod));
        System.out.println("Before five months and one day: " + now.minus(fiveMonthAndOneDayPeriod));

		//        Now: 2021-10-27
		//        After five months and one day: 2022-03-28
		//        Before five months and one day: 2021-05-26
    }
}
```

## Durations

Duration also represents a distance on the timeline. After that we can add or minus amount of duration from dates.

**Duration distance is based on amount of time, such as '34.5 seconds'**

```java
Duration differenceBetweenTimes = Duration.between(oneLocalTime, anotherLocalTime);
```

## Date format on the the new date APIs

We can use `DateTimeFormatter`.

```java
public class Test {

    public static void main(String[] args) {
        LocalDate now = LocalDate.now();
        String formatted = now.format(DateTimeFormatter.ofPattern("yyyy/MM/dd"));
        System.out.println("Now: " + now);
        System.out.println("Formatted: " + formatted);
		// Now: 2021-10-27
		// Formatted: 2021/10/27

        LocalDateTime localDateTime = LocalDateTime.now();
        String anotherFormat = localDateTime.format(DateTimeFormatter.ofPattern("yyyy.MM.dd"));
		String removeNanoSec = localDateTime.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        System.out.println("LocalDateTime now: " + localDateTime);
        System.out.println("Another format: " + anotherFormat);
        System.out.println("Without nano seconds: " + removeNanoSec);

        //        LocalDateTime now: 2021-10-27T23:43:06.672018
		//        Another format: 2021.10.27
		//        Without nano seconds: 2021-10-27 23:43:06
    }
}
```

## Instant

Instant represents single specific point (instantaneous point) on the time line in UTC (a count of nanoseconds since the epoch of the first moment of 1970 UTC.)

This class can be useful to record event-time in the application. And Instant has nanoseconds precision.

Here are the some important methods:

- `Instant.now()` returns current instant
- `Instant.ofEpochMilli(long epochMilli)`: returns Instant using milliseconds from the epoch of 1970-01-01T00:00:00Z.
- `Instant.ofEpochSecond()`: returns Instant using seconds from the epoch of 1970-01-01T00:00:00Z.

- `Instant now().toEpochMilli()`: Converts this instant to the number of milliseconds from the epoch of 1970-01-01T00:00:00Z.
- `Instant.now().getEpochSecond()`: Gets the number of seconds from the Java epoch of 1970-01-01T00:00:00Z.

```java
public class Test {

    public static void main(String[] args) {
        Instant now = Instant.now();
        System.out.println("Instant: " + now);
        System.out.println("Instant.toEpochMilli(): " + now.toEpochMilli());
        System.out.println("Instant.getEpochSecond(): " + now.getEpochSecond());
        //        Instant: 2021-10-27T21:02:47.803400Z
		//        Instant.toEpochMilli(): 1635368567803
		//        Instant.getEpochSecond(): 1635368567
    }
}
```

### Convert Instant to ZonedDateTime

We can use `ofInstant()`:

```java
public class Test {

    public static void main(String[] args) {
        Instant now = Instant.now();
        ZonedDateTime fromInstant = ZonedDateTime.ofInstant(now, ZoneId.systemDefault());
        System.out.println("Instant: " + now);
        System.out.println("ZonedDateTime: " + fromInstant);
        // Instant: 2021-10-27T21:05:53.059012Z
		// ZonedDateTime: 2021-10-28T00:05:53.059012+03:00[Europe/Istanbul]
    }
}
```

### Convert Instant to LocalDateTime

Because Instant implicitly refers to the UTC time line, we must also use ZoneId while converting to the LocalDateTime

> Or we can convert ZonedDateTime to LocalDateTime:
>
> ```java
> ZonedDateTime.ofInstant(now, ZoneId.systemDefault()).toLocalDateTime();
> ```

```java
public class Test {

    public static void main(String[] args) {
        Instant now = Instant.now();
        LocalDateTime localDateTime = LocalDateTime.ofInstant(now, ZoneId.systemDefault());
        ZonedDateTime fromInstant = ZonedDateTime.ofInstant(now, ZoneId.systemDefault());
        System.out.println("Instant: " + now);
        System.out.println("LocalDateTime: " + localDateTime);
        System.out.println("ZonedDateTime: " + fromInstant);
//        Instant: 2021-10-27T21:12:36.344860Z
//        LocalDateTime: 2021-10-28T00:12:36.344860
//        ZonedDateTime: 2021-10-28T00:12:36.344860+03:00[Europe/Istanbul]

    }
}
```

## Working with JPA and PostgreSQL

JPA 2.2 supports the following classes:

```java
java.time.LocalDate
java.time.LocalTime
java.time.LocalDateTime
java.time.OffsetTime
java.time.OffsetDateTime
```

Also PostgreSQL JDBC driver implements native support for the Java 8 Date and Time API

For more you can refer my blog => [https://mehmetozanguven.github.io/jpa/2021/08/06/jpa-fundamentals-and-hibernate-enumarated-temporal-and-date-types.html#working-with-dates](https://mehmetozanguven.github.io/jpa/2021/08/06/jpa-fundamentals-and-hibernate-enumarated-temporal-and-date-types.html#working-with-dates)

## Conclusion

The new date API provides more concrete and easy way to manage dates in Java.

In the new date API, Local means (LocalDate, LocalTime etc..) there is no zone information in it.

If you want to comply with DST rules, in other words, if you application also need to change UTC offset in the summer time, then you must use **ZonedDateTime** with **ZoneID**.

If you want to stick with only offset from UTC, then you must use **OffsetDateTime** with **ZoneOffset**

In database level, you should use **OffsetDateTime**, because ZonedDateTime may lead to bug(s) or even income failure(s).

Finally JPA 2.2 supports the following date classes for database operation(s):

```wiki
java.time.LocalDate
java.time.LocalTime
java.time.LocalDateTime
java.time.OffsetTime
java.time.OffsetDateTime
```
