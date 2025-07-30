#  양방향 연관관계와 영속성 관리

오늘은 JPA의 핵심 개념인 **양방향 연관관계** 설정, **영속성 전이(Cascade)**와 **고아 객체(Orphan Removal)**, 그리고 성능 최적화를 위한 **N+1 문제 해결** 방법에 대해 학습했습니다.

---

## 1. 다대다(N:M) 관계의 해소

데이터베이스에서 **다대다(N:M) 관계**는 직접 표현하기보다는 중간에 **연결 테이블(Junction Table)**을 두어 두 개의 **일대다(1:N)** 관계로 풀어내는 것이 일반적입니다.

예를 들어, 한 명의 **회원(`User`)**이 여러 개의 **상품(`Goods`)**을 구매할 수 있고, 하나의 상품은 여러 회원에 의해 구매될 수 있는 다대다 관계가 있습니다. 이를 `Purchase`라는 연결 엔티티를 통해 해소할 수 있습니다.

-   **관계 정의**
    -   `User` ↔ `Purchase` : 일대다 (1:N)
    -   `Goods` ↔ `Purchase` : 일대다 (1:N)
    -   결과적으로 `Purchase`는 `User`와 `Goods`에 대해 **다대일(N:1)** 관계를 가집니다.

### 📝 코드 예시

**`Purchase.java` (연결 엔티티)**
`@ManyToOne`을 사용해 `User`와 `Goods`와의 관계를 정의합니다. `Purchase` 테이블이 각 `user_id`와 `goods_id` 외래 키를 가지게 됩니다.

```java
package com.spring.database.jpa.chap04.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "tbl_mtm_purchase")
@Getter @Setter @ToString(exclude = {"user", "goods"})
@NoArgsConstructor @AllArgsConstructor @Builder
public class Purchase {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // 다대다 관계 해소를 위해 Purchase는 User, Goods와 각각 다대일 관계를 맺음
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "goods_id")
    private Goods goods;
}
````

**`User.java` (1측 엔티티)**
`@OneToMany`를 통해 `Purchase`와의 일대다 관계를 설정합니다. `mappedBy = "user"`는 이 관계가 `Purchase` 엔티티의 `user` 필드에 의해 관리된다는 것을 의미합니다. (연관관계의 주인이 아님)

```java
// ...
@Entity
@Table(name = "tbl_mtm_user")
public class User {
    // ...
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    List<Purchase> purchases = new ArrayList<>();
}
```

-----

## 2\. 양방향 연관관계와 편의 메서드

**양방향 연관관계**는 두 엔티티가 서로를 참조하는 관계입니다. 이때, 두 엔티티 중 하나를 \*\*연관관계의 주인(Owner)\*\*으로 정해야 합니다.

  - **연관관계의 주인**: 외래 키(FK)를 관리하는 엔티티입니다. 보통 다(N)측이 주인이 되며, `@JoinColumn` 어노테이션을 사용합니다.
  - **주인이 아닌 측**: `mappedBy` 속성을 사용해 주인을 명시합니다. 이 측에서는 외래 키를 변경할 수 없으며, 오직 조회만 가능합니다.

### 💡 편의 메서드의 중요성

양방향 관계에서 한쪽 객체의 참조만 설정하면 다른 쪽 객체에는 반영되지 않아 객체 상태의 불일치가 발생할 수 있습니다. 이를 방지하기 위해 **편의 메서드**를 만들어 양쪽의 참조를 항상 함께 설정하는 것이 좋습니다.

```java
// Department 엔티티 내 편의 메서드 예시
public void addEmployee(Employee employee) {
    this.employees.add(employee); // 1. 부서의 사원 리스트에 사원 추가
    employee.setDepartment(this); // 2. 사원 객체에도 현재 부서를 설정
}
```

아래 `persistTest`는 편의 메서드를 사용하여 `department`의 `employees` 리스트에 `employee`를 추가하는 것만으로 `employee`의 `department` 필드까지 설정되어 DB에 정상적으로 INSERT 되는 것을 보여줍니다.

```java
@Test
@DisplayName("양방향 매핑 리스트에서 사원을 추가하면 DB에도 INSERT된다.")
void persistTest() {
    //given
    Department department = departmentRepository.findById(2L).orElseThrow();
    Employee employee = Employee.builder().name("파이리").build();

    //when
    // 편의 메서드를 사용해 양쪽 객체에 관계를 설정
    department.addEmployee(employee);

    em.flush();
    em.clear();

    //then
    Employee foundEmp = employeeRepository.findById(5L).orElseThrow();
    assertEquals("파이리", foundEmp.getName());
    assertEquals(2L, foundEmp.getDepartment().getId());
}
```

-----

## 3\. 영속성 전이(Cascade)와 고아 객체 제거(Orphan Removal)

### `CascadeType` (영속성 전이)

부모 엔티티의 영속성 상태 변화를 자식 엔티티에게도 전파하는 기능입니다. 예를 들어, 부모를 저장하면 자식도 함께 저장되고, 부모를 삭제하면 자식도 함께 삭제됩니다.

| CascadeType | 설명                                                              |
| :---------- | :---------------------------------------------------------------- |
| `ALL`       | 모든 영속성 상태 변경(`PERSIST`, `MERGE`, `REMOVE` 등)을 전파합니다. |
| `PERSIST`   | 부모를 저장(`persist`)할 때 자식도 함께 저장됩니다.                |
| `MERGE`     | 부모를 병합(`merge`)할 때 자식도 함께 병합됩니다.                  |
| `REMOVE`    | 부모를 삭제(`remove`)할 때 자식도 함께 삭제됩니다.                |

**`cascadeTest` 예제**
`User` 엔티티의 `purchases` 필드에 `cascade = CascadeType.ALL`이 설정되어 있습니다. 따라서 `user1`을 삭제하면, `user1`이 가지고 있던 모든 `Purchase` 기록도 함께 삭제됩니다.

```java
@Test
@DisplayName("회원이 탈퇴하면 구매기록이 모두 사라져야한다.")
void cascadeTest () {
    // ... (user1에게 여러 구매 기록을 생성)

    // when: user1을 조회하고 삭제
    User user = userRepository.findById(1L).orElseThrow();
    userRepository.delete(user);

    em.flush();
    em.clear();

    // then: user1의 구매 기록이 DB에서 사라졌는지 확인
    List<Purchase> purchaseList = purchaseRepository.findAll();
    // user1과 관련된 구매 기록은 모두 삭제되고 user2의 구매 기록만 남는다.
    assertEquals(1, purchaseList.size());
}
```

### `orphanRemoval = true` (고아 객체 제거)

부모 엔티티와의 연관관계가 끊어진 자식 엔티티를 자동으로 삭제하는 기능입니다. `CascadeType.REMOVE`는 부모가 삭제될 때 자식이 삭제되지만, `orphanRemoval`은 부모의 컬렉션에서 자식을 **제거**하는 것만으로도 자식의 `DELETE` 쿼리가 실행됩니다.

**`orphanRemovalTest` 예제**
`Department`의 `employees` 컬렉션에서 특정 `Employee`를 `remove()` 하면, 해당 `Employee`는 '고아'로 취급되어 DB에서 자동으로 삭제됩니다.

```java
@Test
@DisplayName("양방향 매핑 리스트에서 사원을 제거하면 실제 DB에서 DELETE된다.")
void orphanRemovalTest() {
    //given
    // 1번 부서와 소속 사원들을 조회
    Department foundDept = departmentRepository.findById(1L).orElseThrow();
    List<Employee> employees = foundDept.getEmployees();

    //when
    // 부서의 사원 리스트에서 첫 번째 사원을 제거
    employees.remove(0);

    // em.flush(), em.clear() 실행 후 확인하면
    // 해당 사원은 DB에서 DELETE 되어있음.
}
```

-----

## 4\. N+1 문제와 Fetch Join

### 😵 N+1 문제란?

연관관계가 있는 엔티티를 조회할 때 발생하는 대표적인 성능 문제입니다.

1.  먼저 부모 엔티티 목록을 조회하는 쿼리 1개가 실행됩니다. (**1**)
2.  이후 각 부모 엔티티에 대해 연관된 자식 엔티티를 조회하기 위해 N개의 추가 쿼리가 실행됩니다. (**N**)

**`nPlusOneTest` 예제**
`employeeRepository.findAll()`을 호출하면 모든 사원을 조회하는 쿼리(**1**)가 나갑니다. 이후 `forEach` 루프에서 각 사원의 부서 정보(`dept.getDepartment()`)에 접근할 때마다 부서를 조회하는 쿼리가 사원의 수(**N**)만큼 추가로 발생합니다.

```java
@Test
@DisplayName("N + 1 문제 확인")
void nPlusOneTest() {
    //given

    //when
    // 1. 모든 사원을 조회하는 쿼리 1번 발생
    List<Employee> employees = employeeRepository.findAll();

    //then
    employees.forEach(emp -> {
        // 2. 루프를 돌면서 각 사원의 부서 정보를 얻기 위해 쿼리가 N번 추가 발생
        System.out.println("사원명: " + emp.getName() + ", 부서명: " + emp.getDepartment().getName());
    });
}
```

### ✨ 해결 방안: Fetch Join

**Fetch Join**은 JPQL을 통해 엔티티를 조회할 때, 연관된 엔티티를 **즉시 함께 조회**하도록 하는 기능입니다. SQL의 `JOIN`을 사용하여 처음부터 필요한 모든 데이터를 가져오므로 N+1 문제가 발생하지 않습니다.

**`fetchJoinTest` 예제**
`Department` 리포지토리에 `JOIN FETCH`를 사용한 JPQL 쿼리를 작성합니다.

```java
// DepartmentRepository.java
@Query("SELECT DISTINCT d FROM Department d JOIN FETCH d.employees")
List<Department> findAllByFetch();
```

이제 `findAllByFetch()`를 호출하면 단 한 번의 쿼리로 모든 부서와 각 부서에 속한 사원 정보까지 함께 가져옵니다.

```java
@Test
@DisplayName("N + 1 문제 해결을 위한 fetch join")
void fetchJoinTest() {
    //given

    //when
    // JOIN FETCH가 적용된 쿼리를 통해 단 1번의 쿼리만 발생
    List<Department> departments = departmentRepository.findAllByFetch();

    //then
    departments.forEach(d -> {
        System.out.println("부서명: " + d.getName() + ", 소속 사원: " + d.getEmployees().get(0).getName());
    });
}
```
