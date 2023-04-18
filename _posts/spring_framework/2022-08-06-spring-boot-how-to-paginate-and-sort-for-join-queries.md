---
layout: post
title: How to paginate and sort for join queries in Spring Boot
date: 2022-08-06 23:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/spring-boot/spring-boot-how-to-paginate-and-sort-for-join-queries/"
---

Let's say you have two entities which has many-to-one relationship and you want to paginate your query on the parent side with additional colums from child entity.

> For instance you want to paginate and sort your sql queries (for parent side) according to total number of child entities.

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

## Github link

If you only need to see the code, here is the [github link](https://github.com/mehmetozanguven/spring-boot-examples/tree/master/paginate-left-join-queries)

---

<br />

First please create new spring boot project with the following dependencies:

```xml
<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-data-jpa</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
    <dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<optional>true</optional>
		</dependency>
	<dependency>
		<groupId>com.h2database</groupId>
		<artifactId>h2</artifactId>
		<scope>runtime</scope>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
</dependencies>
```

## Creating entities

For the basic project, we will have School entity and Student entity. And our relationship between Student and School is the many-to-one. (Many students can belong to the one School)

School entity:

```java
@Entity
@Table(name = "school")
@Builder
@AllArgsConstructor
@NoArgsConstructor
@Data
public class School {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column
    private String name;
}
```

Student entity:

```java
@Entity
@Table(name = "student")
@Builder
@AllArgsConstructor
@NoArgsConstructor
@Data
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column
    private String name;

    @ManyToOne
    @JoinColumn(name = "school_id")
    @OnDelete(action = OnDeleteAction.CASCADE)
    private School school;
}
```

## Creating repositories and services

Let's also create basic related repository and service for the entities:

School repository and service:

```java
@Repository
@Transactional(readOnly = true)
public interface SchoolRepository extends JpaRepository<School, Long> {
}
```

```java
@Service
@RequiredArgsConstructor
public class SchoolService {
    private final SchoolRepository schoolRepository;

    @Transactional
    public void createSchool(School school) {
        schoolRepository.save(school);
    }

    public Page<School> findSchoolsInPage(Pageable pageable) {
        return schoolRepository.findAll(pageable);
    }
}
```

Student repository and service:

```java
@Repository
@Transactional(readOnly = true)
public interface StudentRepository extends JpaRepository<Student, Long> {

}
```

```java
@Service
@RequiredArgsConstructor
public class StudentService {
    private final StudentRepository studentRepository;

    public Student findById(Long id) {
        return studentRepository.findById(id).orElseThrow();
    }

    @Transactional
    public void createStudent(Student student) {
        studentRepository.save(student);
    }
}
```

## Creating controller

We have only one controller:

```java
@RestController
@RequestMapping(value = "/api")
@RequiredArgsConstructor
public class SchoolRestController {
    private final SchoolService schoolService;

    @GetMapping(value = "/find")
    public ResponseEntity<Page<School>> findSchoolsInPage(Pageable pageable) {
        Page<School> response = schoolService.findSchoolsInPage(pageable);
        return new ResponseEntity<>(response, HttpStatus.OK);
    }
}
```

## Initialize the database

For simplicity, I am just creating random objects in the main method:

```java
@SpringBootApplication
@RequiredArgsConstructor
public class PaginateLeftJoinQueriesApplication {
	private final SchoolRepository schoolRepository;
	private final StudentRepository studentRepository;

	public static void main(String[] args) {
		SpringApplication.run(PaginateLeftJoinQueriesApplication.class, args);
	}

	@PostConstruct
	public void initDatabase() {
		int schoolCount = RandomUtils.nextInt(20, 100);
		for (int index = 0; index < schoolCount; index++) {
			School school = School
					.builder()
					.name(RandomStringUtils.randomAlphabetic(10))
					.build();
			school = schoolRepository.save(school);
			int studentCount = RandomUtils.nextInt(5, 50);
			for (int studentIndex = 0; studentIndex < studentCount; studentIndex++) {
				Student student = Student
						.builder()
						.name(RandomStringUtils.randomAlphabetic(10))
						.school(school)
						.build();
				studentRepository.save(student);
			}
		}
	}
}
```

Now let's talk about (complex!) queries and paginate them.

## Complex queries with pagination and sorting

Let's assume that we want to get all school entries with total number of student and also we want to use paginate function within Spring framework.

Because we want to get the total number of student which is extra column, we can't do that in the School entity. Fortunately JPA provides nice feature:

In the query, we can return new object and this object doesn't need to be entity. Let's create class for the object:

```java
@AllArgsConstructor
@Data
@ToString
public class SchoolWithStudentCount {
    private School school;
    private long studentCount;
}
```

This is very simple class which includes total number of student in the school. But because we are mainly dealing with School and because SchoolWithStudentCount is not entity we can't create new repository like `public interface SchoolRepository extends JpaRepository<SchoolWithStudentCount, Long>`, this will give us an exception.

Instead of creating new repository, we can add query method in the SchoolRepository:

```java
@Repository
@Transactional(readOnly = true)
public interface SchoolRepository extends JpaRepository<School, Long> {
    @Query(
            value = "SELECT new com.mehmetozanguven.paginateleftjoinqueries.entity.SchoolWithStudentCount" +
                    "(" +
                    "sc, " +
                    "COUNT(st.id) AS studentCount" +
                    ") " +
                    "FROM School sc  " +
                    "LEFT JOIN Student st ON st.school = sc " +
                    "GROUP BY sc"
    )
    Page<SchoolWithStudentCount> findSchoolWithStudentCount(Pageable pageable);
}
```

In the part **new com.mehmetozanguven.paginateleftjoinqueries.entity** we are actually returning new object. And the best part is that **we can sort the result with the properties from School object or new property called studentCount**

The rest is just defining related methods in the service and controller:

```java
@Service
@RequiredArgsConstructor
public class SchoolService {
    private final SchoolRepository schoolRepository;

    @Transactional
    public void createSchool(School school) {
        schoolRepository.save(school);
    }

    public Page<School> findSchoolsInPage(Pageable pageable) {
        return schoolRepository.findAll(pageable);
    }

    public Page<SchoolWithStudentCount> findSchoolsWithStudentCountInPage(Pageable pageable) {
        return schoolRepository.findSchoolWithStudentCount(pageable);
    }
}
```

```java

@RestController
@RequestMapping(value = "/api")
@RequiredArgsConstructor
public class SchoolRestController {
    private final SchoolService schoolService;


    @GetMapping(value = "/find")
    public ResponseEntity<Page<School>> findSchoolsInPage(Pageable pageable) {
        Page<School> response = schoolService.findSchoolsInPage(pageable);
        return new ResponseEntity<>(response, HttpStatus.OK);
    }

    @GetMapping(value = "/find-with-student-count")
    public ResponseEntity<Page<SchoolWithStudentCount>> findSchoolsWithStudentCountInPage(Pageable pageable) {
        Page<SchoolWithStudentCount> response =schoolService.findSchoolsWithStudentCountInPage(pageable);
        return new ResponseEntity<>(response, HttpStatus.OK);
    }
}
```

Now after running the project please send the following request from your terminal or browser:

`http://localhost:8080/api/find-with-student-count?page=0&size=20&sort=studentCount,desc&sort=name,desc`

You will see that response will be sorted by studentCount and School's name

```json
{
  "content": [
    {
      "school": {
        "id": 5,
        "name": "KuXpEtgonI"
      },
      "studentCount": 49
    },
    {
      "school": {
        "id": 6,
        "name": "VmOUdaEFmc"
      },
      "studentCount": 48
    },
    {
      "school": {
        "id": 3,
        "name": "DqBFZzJWkD"
      },
      "studentCount": 44
    }
    // ...
  ],
  "pageable": {
    "sort": {
      "empty": false,
      "sorted": true,
      "unsorted": false
    },
    "offset": 0,
    "pageNumber": 0,
    "pageSize": 20,
    "paged": true,
    "unpaged": false
  },
  "last": true,
  "totalPages": 1,
  "totalElements": 20,
  "size": 20,
  "number": 0,
  "sort": {
    "empty": false,
    "sorted": true,
    "unsorted": false
  }
  // ...
}
```

We can also return School entities with the Student's name as well. Steps will be similar to the previous one:

- Create new object for the query which accepts the related entities, properties in the constructor
- Write the query with "new ..." keyword and pass the entities, properties in the repository
- Return the response

For the student's name, I have just created new class:

```java
@AllArgsConstructor
@Data
@ToString
public class SchoolWithStudentName {
    private School school;
    private String studentName;
}
```

Update the repository:

```java
@Repository
@Transactional(readOnly = true)
public interface SchoolRepository extends JpaRepository<School, Long> {
    // ...

    @Query(
            value = "SELECT new com.mehmetozanguven.paginateleftjoinqueries.entity.SchoolWithStudentName" +
                    "(" +
                    "sc, " +
                    "st.name AS studentName" +
                    ") " +
                    "FROM School sc  " +
                    "LEFT JOIN Student st ON st.school = sc "
    )
    Page<SchoolWithStudentName> findSchoolWithStudentName(Pageable pageable);
}
```

Finally update the service and controller:

```java
@Service
@RequiredArgsConstructor
public class SchoolService {
    private final SchoolRepository schoolRepository;

    // ...

    public Page<SchoolWithStudentName> findSchoolWithStudentName(Pageable pageable) {
        return schoolRepository.findSchoolWithStudentName(pageable);
    }
}
```

```java
@RestController
@RequestMapping(value = "/api")
@RequiredArgsConstructor
public class SchoolRestController {
    private final SchoolService schoolService;

    // ....

    @GetMapping(value = "/find-with-student-name")
    public ResponseEntity<Page<SchoolWithStudentName>> findSchoolWithStudentName(Pageable pageable) {
        Page<SchoolWithStudentName> response = schoolService.findSchoolWithStudentName(pageable);
        return new ResponseEntity<>(response, HttpStatus.OK);
    }
}
```

Then following request will return a response which is sorted by studentName:

`http://localhost:8080/api/find-with-student-name?page=0&size=20&sort=studentName,desc`

```json
{
  "content": [
    {
      "school": {
        "id": 12,
        "name": "QLgFunugzc"
      },
      "studentName": "zyHmIlvEFY"
    },
    {
      "school": {
        "id": 14,
        "name": "DxoYdzFzRx"
      },
      "studentName": "zsjfLZjHSD"
    },
    {
      "school": {
        "id": 4,
        "name": "WBoTYIlUTq"
      },
      "studentName": "zQMTzkqhRj"
    }
    // ...
  ],
  "pageable": {
    "sort": {
      "empty": false,
      "sorted": true,
      "unsorted": false
    },
    "offset": 0,
    "pageNumber": 0,
    "pageSize": 20,
    "paged": true,
    "unpaged": false
  },
  "last": false,
  "totalPages": 23,
  "totalElements": 458,
  "size": 20,
  "number": 0,
  "sort": {
    "empty": false,
    "sorted": true,
    "unsorted": false
  },
  "first": true,
  "numberOfElements": 20,
  "empty": false
}
```
