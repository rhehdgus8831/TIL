
# JPA  연관관계

## 1\. JPA 연관관계 매핑 (JPA Relationship Mapping)

객체지향 프로그래밍에서는 객체 간의 관계를, 관계형 데이터베이스에서는 테이블 간의 관계를 맺습니다. JPA는 이 두 패러다임의 불일치를 해결하고, 개발자가 객체 중심으로 프로그래밍할 수 있도록 돕습니다.

### 1.1. 주요 개념: 단방향과 양방향

  * **단방향(Unidirectional) 매핑**: 한쪽 엔티티만 상대 엔티티를 참조하는 관계입니다. 예를 들어, `Employee` 엔티티에서만 `Department` 정보를 참조하는 경우입니다.
  * **양방향(Bidirectional) 매핑**: 양쪽 엔티티가 서로를 참조하는 관계입니다. `Employee`는 `Department`를, `Department`는 소속된 `Employee` 리스트를 참조하는 경우입니다.

> **중요**: 관계의 주인(Owner)
>
> 양방향 관계에서는 외래 키(Foreign Key)를 관리하는 주체를 정해야 합니다. 보통 \*\*N쪽(Many)\*\*이 관계의 주인이 되며, `@JoinColumn` 어노테이션을 사용하여 외래 키를 관리합니다. 주인 쪽의 변경사항만 데이터베이스에 반영됩니다.

-----

### 1.2. 엔티티 코드 분석

#### 📄 `Employee.java` (N쪽, 연관관계의 주인)

`Employee`는 '다(N)'에 해당하며, `Department`와 `@ManyToOne` 관계를 맺습니다. 실제 테이블에서 `dept_id` 외래 키를 관리하므로 연관관계의 주인입니다.

```java
package com.spring.database.jpa.chap03.entity;

import jakarta.persistence.*;
import lombok.*;

@Getter
@Setter
@ToString(exclude = {"department"}) // 순환 참조 방지를 위해 department 필드 제외
@EqualsAndHashCode
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Entity
@Table(name = "tbl_emp")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "emp_id")
    private Long id; // 사원번호

    @Column(name = "emp_name", nullable = false)
    private String name; // 사원명

    // N:1 관계 매핑
    // Employee가 N, Department가 1
    @ManyToOne(fetch = FetchType.LAZY) // 지연 로딩 설정
    @JoinColumn(name = "dept_id") // FK 컬럼명 지정
    private Department department; // 부서정보

    // 양방향 관계에서 데이터 일관성을 위한 편의 메서드
    public void changeDepartment(Department department) {
        this.department = department;
        if (department != null) {
            // 반대편(Department)의 employees 리스트에도 현재 Employee 객체를 추가
            department.getEmployees().add(this);
        }
    }
}
```

#### 📄 `Department.java` (1쪽)

`Department`는 '일(1)'에 해당하며, 자신에게 소속된 `Employee` 목록을 참조하기 위해 `@OneToMany` 관계를 맺습니다. `mappedBy` 속성을 통해 이 관계의 주인이 `Employee` 엔티티의 `department` 필드임을 명시합니다.

```java
package com.spring.database.jpa.chap03.entity;

import jakarta.persistence.*;
import lombok.*;
import java.util.ArrayList;
import java.util.List;

@Getter
@Setter
@ToString(exclude = {"employees"}) // 순환 참조 방지를 위해 employees 필드 제외
@EqualsAndHashCode
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Entity
@Table(name = "tbl_dept")
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "dept_id")
    private Long id; // 부서번호

    @Column(name = "dept_name", nullable = false)
    private String name; // 부서명

    /*
        # 양방향 매핑

        - 데이터베이스와 달리 객체지향 시스템에서 가능한 방법으로,
          1:N 관계에서 1쪽에 N 데이터를 포함시킬 수 있는 방법이다.

        - 양방향 매핑에서 1쪽은 상대방 엔티티 갱신에 관여할 수 없고,
          (리스트에서 사원을 지운다고 실제 DB에서 사원이 삭제되지는 않음)
          단순히 읽기 전용(조회 전용)으로만 사용된다.

        - `mappedBy`에는 상대방 엔티티(Employee)의 @ManyToOne 필드명('department')을 꼭 적어야 한다.
    */
    @OneToMany(mappedBy = "department", fetch = FetchType.EAGER)
    private List<Employee> employees = new ArrayList<>();
}
```

-----

### 1.3. 핵심 어노테이션 및 속성 정리

#### `@ManyToOne` / `@OneToMany`

| 속성 | 설명 |
| :--- | :--- |
| `fetch` | 연관된 엔티티를 조회하는 전략을 설정합니다. (아래 `FetchType` 참조) |
| `mappedBy` | **(`@OneToMany` 전용)** 연관관계의 주인이 아님을 명시합니다. 값으로는 주인 엔티티에 있는 연관관계 필드명을 지정합니다. (`mappedBy="department"`) |
| `cascade` | 영속성 전이 기능을 설정합니다. 부모 엔티티의 상태 변화를 자식 엔티티에게 전파합니다. (`CascadeType.PERSIST`, `CascadeType.REMOVE`, `CascadeType.ALL` 등) |
| `orphanRemoval`| `true`로 설정 시, 부모 엔티티 컬렉션에서 자식 엔티티가 제거되면 데이터베이스에서도 해당 자식을 삭제합니다. (고아 객체 제거) |

#### `FetchType` (조회 전략)

| 타입 | 설명 | 권장 사항 |
| :--- | :--- | :--- |
| `FetchType.EAGER` (즉시 로딩) | 주 엔티티를 조회할 때 연관된 엔티티도 즉시 함께 조회합니다. | **@ManyToOne의 기본값.** N+1 문제를 유발할 수 있어 주의가 필요합니다. |
| `FetchType.LAZY` (지연 로딩) | 주 엔티티 조회 시 연관된 엔티티는 프록시(가짜) 객체로 가져오고, 실제 사용 시점에 조회 쿼리가 실행됩니다. | **@OneToMany의 기본값.** 성능 최적화를 위해 **`@ManyToOne` 관계에서는 항상 LAZY로 설정하는 것을 권장**합니다. |

-----

### 1.4. 연관관계 테스트 코드

JPA 연관관계가 올바르게 설정되었는지 테스트를 통해 확인할 수 있습니다.

#### 📄 `EmployeeRepositoryTest.java`

```java
package com.spring.database.jpa.chap03.repository;

import com.spring.database.jpa.chap03.entity.Department;
import com.spring.database.jpa.chap03.entity.Employee;
import jakarta.persistence.EntityManager;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.annotation.Rollback;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
@Transactional   // 연관관계 매핑 시 LAZY 로딩 등을 위해 트랜잭션 처리가 필수
@Rollback(false) // 테스트 후 데이터 롤백 방지
class EmployeeRepositoryTest {

    @Autowired EmployeeRepository employeeRepository;
    @Autowired DepartmentRepository departmentRepository;
    @Autowired EntityManager em; // 영속성 컨텍스트 관리를 위한 EntityManager

    // 테스트 시작 전 데이터 준비
    @BeforeEach
    void insertBulk() {
        Department d1 = Department.builder().name("영업부").build();
        Department d2 = Department.builder().name("개발부").build();
        departmentRepository.saveAll(List.of(d1, d2));

        Employee e1 = Employee.builder().name("라이언").department(d1).build();
        Employee e2 = Employee.builder().name("어피치").department(d1).build();
        Employee e3 = Employee.builder().name("프로도").department(d2).build();
        Employee e4 = Employee.builder().name("네오").department(d2).build();
        employeeRepository.saveAll(List.of(e1, e2, e3, e4));

        /*
            JPA의 영속성 컨텍스트(Persistence Context)는 엔티티를 보관하는 1차 캐시 역할을 합니다.
            em.flush()는 컨텍스트의 변경 내용을 DB에 동기화하고,
            em.clear()는 컨텍스트를 비워서 이후 조회 시 DB에서 직접 데이터를 가져오도록 합니다.
            (정확한 SELECT 쿼리 확인을 위한 학습용 코드)
        */
        em.flush();
        em.clear();
    }

    @Test
    @DisplayName("특정 사원의 정보를 조회하면 부서정보도 같이 조회된다")
    void findTest() {
        //given
        Long empId = 2L;
        //when
        Employee employee = employeeRepository.findById(empId).orElseThrow();
        //then
        System.out.println("employee = " + employee);

        // FetchType.LAZY 설정 시, department를 실제로 사용하는 이 시점에 SELECT 쿼리가 한 번 더 실행됨
        Department department = employee.getDepartment();
        System.out.println("department = " + department);
    }

    @Test
    @DisplayName("특정 부서를 조회하면 해당 소속 사원들이 함께 조회된다.")
    void FindDeptTest() {
        //given
        Long deptId = 1L;
        //when
        Department foundDept = departmentRepository.findById(deptId).orElseThrow();
        //then
        // FetchType.EAGER 설정 시, 부서 조회 시점에 소속 사원 정보까지 JOIN하여 함께 조회함
        System.out.println("foundDept = " + foundDept);
        System.out.println("foundDept.getEmployees() = " + foundDept.getEmployees());
    }

    @Test
    @DisplayName("양방향 매핑에서 데이터 수정 시 편의 메서드 사용")
    void changeTest() {
        // 3번 사원의 부서를 2번(개발부)에서 1번(영업부)으로 수정

        //given
        Employee foundEmp = employeeRepository.findById(3L).orElseThrow();
        Department newDept = departmentRepository.findById(1L).orElseThrow();

        //when
        // 연관관계 편의 메서드를 사용하여 양쪽의 관계를 한 번에 설정
        foundEmp.changeDepartment(newDept);
        employeeRepository.save(foundEmp);

        em.flush(); // DB 동기화
        em.clear(); // 영속성 컨텍스트 초기화

        //then
        // 변경된 사원의 부서 정보 확인
        Employee updatedEmp = employeeRepository.findById(3L).orElseThrow();
        assertEquals("영업부", updatedEmp.getDepartment().getName());
        System.out.println("\n\n변경 후 사원 정보 = " + updatedEmp + ", 부서: " + updatedEmp.getDepartment());

        // 1번 부서의 사원 정보를 다시 조회 -> 3명이어야 함 (기존 2명 + 이동해온 1명)
        Department updatedDept = departmentRepository.findById(1L).orElseThrow();
        assertEquals(3, updatedDept.getEmployees().size());
        System.out.println("\n\n1번 부서 소속 사원 목록 = " + updatedDept.getEmployees());
    }
}
```

-----

## 2\. JPQL과 네이티브 SQL

Spring Data JPA는 메서드 이름으로 쿼리를 생성하는 기능 외에도, 복잡한 쿼리를 직접 작성할 수 있도록 `@Query` 어노테이션을 제공합니다.

### 2.1. JPQL (Java Persistence Query Language)

  * **객체지향 쿼리 언어**입니다. 테이블이 아닌 **엔티티 객체**를 대상으로 쿼리를 작성합니다.
  * 데이터베이스에 독립적이어서, DB 종류가 변경되어도 쿼리를 수정할 필요가 없습니다.

<!-- end list -->

```java
// JPQL 사용 예시: 도시 이름으로 학생 조회
// Student는 테이블명이 아닌 엔티티 클래스 이름
@Query("SELECT s FROM Student s WHERE s.city = ?1")
List<Student> getStudentByCity(String city);
```

### 2.2. Native SQL (순수 SQL)

  * 데이터베이스에 종속적인 **표준 SQL**을 그대로 사용합니다.
  * 특정 데이터베이스에만 존재하는 함수나 문법을 사용해야 할 때 유용합니다.
  * `nativeQuery = true` 속성을 반드시 설정해야 합니다.

<!-- end list -->

```java
// Native SQL 사용 예시: 도시와 이름으로 학생 조회
@Query(
    value = """
            SELECT *
            FROM tbl_student
            WHERE city = ?1
            AND stu_name = ?2
            """,
    nativeQuery = true
)
List<Student> getStudents(String city, String name);
```

### 2.3. JPQL 및 Native SQL 테스트 코드

```java
@Test
@DisplayName("JPQL로 조회해보기")
void jpqlTest() {
    //given
    String city = "서울시";
    //when
    // Repository에 정의된 JPQL 쿼리 메서드 호출
    List<Student> students = studentRepository.getStudentByCity(city);
    //then
    assertFalse(students.isEmpty());
    students.forEach(System.out::println);
}

@Test
@DisplayName("순수 SQL로 조회하기")
void nativeSqlTest() {
    //given
    String name = "어피치";
    String city = "제주도";
    //when
    // Repository에 정의된 Native SQL 쿼리 메서드 호출
    List<Student> students = studentRepository.getStudents(name, city);
    //then
    assertFalse(students.isEmpty());
    students.forEach(System.out::println);
}
```
