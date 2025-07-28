
# Spring Data JPA

## 1\. Spring JPA 설정 (`application.yml`)

Spring Boot에서 JPA를 사용하기 위한 기본 설정 파일입니다. 데이터베이스 연결 정보, JPA 관련 동작 등을 정의합니다.

```yml
server:
  port: 9001

spring:
  application:
    name: spring-db202507
  # 데이터베이스 연결 정보
  datasource:
    url: jdbc:mariadb://localhost:3306/spring_study # MariaDB 데이터베이스 주소
    username: root        # DB 계정
    password: mariadb     # DB 비밀번호

  # JPA 관련 설정
  jpa:
    hibernate:
      # ddl-auto: 스키마 자동 생성 전략 설정
      # create: 애플리케이션 실행 시점에 테이블을 삭제하고 새로 생성
      # update: 변경된 스키마만 반영
      # validate: 엔티티와 테이블이 정상 매핑되었는지 확인
      # none: 아무것도 하지 않음 (운영 환경에서 권장)
      ddl-auto: create
    properties:
      hibernate:
        # 실행되는 SQL 쿼리를 보기 좋게 포맷팅
        format_sql: true
    # JPA가 사용할 데이터베이스 종류
    database: mysql

# MyBatis 설정 (JPA와 함께 사용 가능)
mybatis:
  mapper-locations: classpath:mappers/**/*.xml
  configuration:
    map-underscore-to-camel-case: true
  type-aliases-package: com.spring.database.chap03


# 로그 레벨 설정
logging:
  level:
    root: INFO
    # 특정 패키지의 로그 레벨을 더 상세하게 설정
    com.spring.database: DEBUG
    # 하이버네이트가 실행하는 SQL을 로그로 확인
    org:
      hibernate:
        SQL: DEBUG
```

### 주요 JPA 설정 요약

| 프로퍼티                      | 설명                                                                                                                                |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `spring.jpa.hibernate.ddl-auto` | **스키마 자동 생성** 전략을 설정합니다. `create`, `update`, `validate`, `none` 등의 옵션이 있으며, 개발 초기에는 `create`를 주로 사용합니다. |
| `spring.jpa.properties.hibernate.format_sql` | JPA가 생성하는 SQL 쿼리를 콘솔에 출력할 때, **가독성 좋게 포맷팅**할지 여부를 결정합니다. (`true`로 설정 시)                          |
| `logging.level.org.hibernate.SQL` | Hibernate가 실행하는 **SQL문을 로그로 확인**할 수 있도록 `DEBUG` 레벨로 설정합니다.                                                    |

\<br/\>

## 2\. JPA Entity: 데이터베이스 테이블과 매핑

Entity는 데이터베이스의 테이블과 직접 매핑되는 Java 클래스입니다. `@Entity` 어노테이션을 붙여 JPA에게 관리 대상임을 알립니다.

### 예제 1: `Product.java`

```java
package com.spring.database.jpa.chap01.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;
import org.springframework.web.bind.annotation.PutMapping;

import java.time.LocalDateTime;

@EqualsAndHashCode
@Getter
@Setter
@ToString
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Entity // 이 클래스는 데이터베이스 테이블과 1:1로 매칭되는 클래스입니다.
@Table(name = "tbl_product") // 연결될 데이터베이스 테이블 명
// 상품정보를 데이터베이스에 관리
public class Product {

    @Id // PK 지정
    @GeneratedValue(strategy = GenerationType.IDENTITY) // auto_increment 자동으로 1씩 증가
    @Column(name = "prod_id")
    private Long id;

    @Column(name = "prod_nm",length = 50, nullable = false)
    private String name; // 상품명

    @Column(name = "prod_price")
    private int price; // 상품 가격

    @CreationTimestamp // INSERT시 자동으로 시간을 저장
    @Column(updatable = false) // 수정 불가
    private LocalDateTime createdAt; // 상품 등록 시간

    @UpdateTimestamp // UPDATE 문 실행 시 자동으로 시간 수정
    private LocalDateTime updatedAt; // 상품 최종 수정 시간

    // 열거형 데이터는 따로 옵션을 안주면 숫자로 저장함.
    // FOOD : 1 FASHION : 2 ELECTRONIC : 3 ...
    @Enumerated(EnumType.STRING) // ENUM을 문자로 입력 (필수)
    private Category category; // 상품 카테고리

    // 수정용 편의 메서드
    public void changeProduct(String newName, int newPrice, Category newCategory) {
        this.name = newName;
        this.price = newPrice;
        this.category = newCategory;
    }

    public enum Category {
        FOOD, FASHION, ELECTRONIC
    }
}
```

### 예제 2: `Student.java`

```java
package com.spring.database.jpa.chap02.entity;

import jakarta.persistence.*;
import lombok.*;

@EqualsAndHashCode
@Getter
// @Setter 객체의 불변성으로 인해 X
@ToString
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Entity
@Table(name = "tbl_student")
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID) // 범용 고유 식별자(UUID)를 ID로 사용
    private String id; // PK를 String으로

    @Column(name = "stu_name",nullable = false)
    private String name; // 학생이름

    private String city;

    private String major;
}
```

### Entity 관련 주요 어노테이션

| 어노테이션 | 설명 | 사용법 및 파라미터 |
| :--- | :--- | :--- |
| `@Entity` | 클래스를 JPA 엔티티로 지정합니다. | 클래스 선언부에 작성 |
| `@Table` | 엔티티와 매핑될 테이블 정보를 설정합니다. | `name`: 테이블 이름 |
| `@Id` | 해당 필드를 테이블의 기본 키(PK)로 지정합니다. | PK로 사용할 필드 위에 작성 |
| `@GeneratedValue` | 기본 키 값을 자동으로 생성하는 전략을 설정합니다. | `strategy`: `IDENTITY`(Auto-increment), `SEQUENCE`, `TABLE`, `UUID` 등 |
| `@Column` | 필드와 매핑될 테이블의 컬럼 정보를 설정합니다. | `name`: 컬럼 이름\<br/\>`nullable`: `false`로 설정 시 `NOT NULL` 제약조건\<br/\>`length`: 문자열 길이\<br/\>`updatable`: `false`로 설정 시 수정 불가 |
| `@CreationTimestamp` | 엔티티가 **생성될 때**의 시간을 자동으로 저장합니다. | `LocalDateTime` 또는 `Timestamp` 타입의 필드에 사용 |
| `@UpdateTimestamp` | 엔티티가 **수정될 때**마다 시간을 자동으로 저장합니다. | `LocalDateTime` 또는 `Timestamp` 타입의 필드에 사용 |
| `@Enumerated` | Enum 타입을 데이터베이스에 어떻게 저장할지 결정합니다. | `EnumType.STRING`: Enum의 이름을 문자열로 저장 (권장)\<br/\>`EnumType.ORDINAL`: Enum의 순서를 숫자로 저장 (기본값, 비권장) |

\<br/\>

## 3\. JpaRepository: 데이터베이스 조작 인터페이스

`JpaRepository` 인터페이스를 상속받아 간단한 인터페이스를 정의하는 것만으로도 기본적인 CRUD(Create, Read, Update, Delete) 및 페이징 처리가 가능합니다.

```java
// JpaRepository<첫번째: 엔터티 클래스, 두번째: 엔터티의 ID 필드 타입>
public interface ProductJpaRepository extends JpaRepository<Product, Long> {

}
```

  - **ProductJpaRepository.java**
    ```java
    package com.spring.database.jpa.chap01.repository;

    import com.spring.database.jpa.chap01.entity.Product;
    import org.springframework.data.jpa.repository.JpaRepository;

    public interface ProductJpaRepository extends JpaRepository<Product, Long> {

    }
    ```
  - **StudentRepository.java**
    ```java
    package com.spring.database.jpa.chap02;

    import com.spring.database.jpa.chap02.entity.Student;
    import org.springframework.data.jpa.repository.JpaRepository;

    import java.util.List;

    // JpaRepository의 제네릭에는 첫번째 엔티티, 두번째 ID 타입
    public interface StudentRepository extends JpaRepository<Student,String>{
        // ... (쿼리 메서드는 아래에서 설명)
    }
    ```

\<br/\>

## 4\. JpaRepository를 이용한 CRUD 테스트

`JpaRepository`가 기본으로 제공하는 메서드를 활용하여 데이터를 조작하는 테스트 코드입니다.

  - **productJpaRepositoryTest.java**

<!-- end list -->

```java
package com.spring.database.jpa.chap01.repository;

import com.spring.database.jpa.chap01.entity.Product;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.annotation.Rollback;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

import static com.spring.database.jpa.chap01.entity.Product.Category.*;
import static org.junit.jupiter.api.Assertions.*;


@SpringBootTest // 스프링이 관리하는 모든 Bean을 로딩하여 테스트 환경을 구축
class ProductJpaRepositoryTest {

    @Autowired
    ProductJpaRepository productJpaRepository;

    // 각 테스트가 실행되기 전에 자동으로 실행될 코드
    @BeforeEach
    void insertBefore(){
        Product p1 = Product.builder()
                .name("아이폰")
                .category(ELECTRONIC)
                .price(2000000)
                .build();
        Product p2 = Product.builder()
                .name("탕수육")
                .category(FOOD)
                .price(20000)
                .build();
        Product p3 = Product.builder()
                .name("구두")
                .category(FASHION)
                .price(300000)
                .build();
        Product p4 = Product.builder()
                .name("주먹밥")
                .category(FOOD)
                .price(1500)
                .build();

        // 전부 insert 하고 commit 까지 (확정)
        productJpaRepository.saveAllAndFlush(
                List.of(p1,p2,p3,p4)
        );
    }

    @Test
    @DisplayName("상품정보를 데이터베이스에 저장한다.")
    void saveTest() {
        //given: 테스트에 사용할 데이터
        Product newProduct = Product.builder()
                .name("정장재킷")
                .price(300000)
                .category(FASHION)
                .build();
        //when: 실제 테스트 동작
        Product saved = productJpaRepository.save(newProduct);

        //then: 테스트 결과 단언
        assertNotNull(saved);
    }


    @Test
    @DisplayName("1번 상품을 삭제한다")
    void deleteTest() {
        //given
        Long id = 1L;
        //when
        productJpaRepository.deleteById(id);
        //then: 삭제 후 조회를 시도하면 결과가 없어야 하지만, 여기서는 예외 발생 여부 등으로 검증 가능
        Optional<Product> product = productJpaRepository.findById(id);
        assertFalse(product.isPresent());
    }

    @Test
    @DisplayName("3번 상품을 단일 조회하면 그 상품명은 구두이다.")
    void findOneTest() {
        //given
        Long id = 3L;

        //when: findById는 Optional을 반환. 값이 없을 수도 있다는 의미.
        Optional<Product> foundProduct = productJpaRepository.findById(id);

        //then: Optional에서 값을 꺼내 확인
        assertTrue(foundProduct.isPresent()); // 값이 존재하는지 확인
        foundProduct.ifPresent(p -> {
            assertEquals("구두", p.getName());
        });

        System.out.println("상품명 = " + foundProduct.map(Product::getName).orElse("상품명없음"));
    }

    @Test
    @DisplayName("2번 상품의 이름과 가격을 수정한다")
    void updateTest() {
        //given
        Long id = 2L;
        String newName = "청소기";
        int newPrice = 150000;
        Product.Category newCategory = ELECTRONIC;

        //when
        /*
             JPA에서는 수정메서드를 따로 제공하지 않습니다.
            단일 조회를 수행한 후 setter를 통해 값을 변경하고
            다시 save를 하면 INSERT문 대신에 UPDATE문이 나갑니다.
            (조회된 엔티티는 JPA가 관리하므로, 트랜잭션 내에서 값 변경만으로도 UPDATE가 실행됨 - '더티 체킹')
        */
        Product foundProduct = productJpaRepository.findById(id).orElseThrow();

        // 수정용 편의 메서드 사용 권장
        foundProduct.changeProduct(newName,newPrice,newCategory);

        productJpaRepository.save(foundProduct); // 변경된 내용을 DB에 최종 반영

        //then: 다시 조회해서 변경 내용을 확인
        Product updatedProduct = productJpaRepository.findById(id).orElseThrow();
        assertEquals(newName, updatedProduct.getName());
    }

    @Test
    @DisplayName("상품을 전체조회하면 총 4개의 상품이 조회된다.")
    void findAllTest() {
        //given

        //when
        List<Product> productList = productJpaRepository.findAll();

        //then
        productList.forEach(System.out::println);
        assertEquals(4,productList.size());
    }
}
```

### 테스트 관련 주요 어노테이션

| 어노테이션 | 설명 |
| :--- | :--- |
| `@SpringBootTest` | Spring Boot 애플리케이션 테스트에 필요한 거의 모든 의존성을 로드합니다. 통합 테스트에 사용됩니다. |
| `@DataJpaTest` | JPA 관련 설정만 로드하여 테스트합니다. `@Repository` 빈만 로딩하며, 기본적으로 인메모리 DB를 사용하고 테스트 후 롤백합니다. |
| `@Transactional` | 테스트 환경에서 이 어노테이션은 테스트 완료 후 모든 데이터 변경사항을 **자동으로 롤백**시킵니다. |
| `@Rollback(false)` | `@Transactional`의 롤백 기능을 비활성화합니다. 데이터베이스에 변경사항이 실제 커밋됩니다. |
| `@BeforeEach` | 각각의 `@Test` 메서드가 실행되기 **전에** 매번 실행될 코드를 지정합니다. |
| `@DisplayName` | JUnit5에서 테스트의 이름을 지정하는 어노테이션입니다. |

\<br/\>

## 5\. 쿼리 메서드 (Query Methods)

`JpaRepository`는 **메서드 이름을 분석하여 자동으로 SQL 쿼리를 생성**하는 강력한 기능을 제공합니다. 정해진 명명 규칙에 따라 메서드를 작성하기만 하면 됩니다.

  - **StudentRepository.java (쿼리 메서드 정의)**

<!-- end list -->

```java
package com.spring.database.jpa.chap02;

import com.spring.database.jpa.chap02.entity.Student;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface StudentRepository extends JpaRepository<Student,String>{

    // Query Method : 메서드에 특별한 이름규칙을 사용해서 SQL을 생성

    // [규칙] findBy + 필드명
    // SELECT * FROM tbl_student WHERE stu_name = ?
    List<Student> findByName(String name);

    // WHERE city = ? AND major = ?
    List<Student> findByCityAndMajor(String city, String major);

    // WHERE major LIKE '%?%'
    List<Student> findByMajorContaining(String major);

    // WHERE major LIKE '?%'
    List<Student> findByMajorStartingWith(String major);

    // WHERE major LIKE '%?'
    List<Student> findByMajorEndingWith(String major);

    // WHERE age <= ?
    // List<Student> findByAgeLessThanEqual(int age);
}
```

  - **StudentRepositoryTest.java (쿼리 메서드 사용)**

<!-- end list -->

```java
@SpringBootTest
class StudentRepositoryTest {

    @Autowired StudentRepository studentRepository;

    @BeforeEach
    void bulkSave(){
        // 테스트 데이터 준비
        Student s1 = Student.builder().name("쿠로미").city("청양군").major("경제학").build();
        Student s2 = Student.builder().name("춘식이").city("서울시").major("컴퓨터공학").build();
        Student s3 = Student.builder().name("어피치").city("제주도").major("화학공학").build();
        studentRepository.saveAllAndFlush(List.of(s1,s2,s3));
    }

    @Test
    @DisplayName("이름이 춘식이인 학생의 모든 정보를 조회한다.")
    void test1() {
        //given
        String name = "춘식이";
        //when
        List<Student> students = studentRepository.findByName(name);
        //then
        assertEquals(1,students.size());
        assertEquals("컴퓨터공학",students.get(0).getMajor());
        System.out.println("students = " + students.get(0));
    }

    @Test
    @DisplayName("도시명과 전공명으로 조회")
    void test2() {
        //given
        String city = "제주도";
        String major = "화학공학";
        //when
        // WHERE city = ? AND major = ?
        List<Student> students = studentRepository.findByCityAndMajor(city, major);
        //then
        assertEquals(1, students.size());
        students.forEach(System.out::println);
    }

    @Test
    @DisplayName("전공에 '공학'이 포함된 학생들 조회")
    void test3() {
        //given
        String containingMajor = "공학";
        //when
        // WHERE major LIKE '%공학%'
        List<Student> students = studentRepository.findByMajorContaining(containingMajor);
        //then
        assertEquals(2, students.size());
        students.forEach(System.out::println);
    }
}
```

### 쿼리 메서드 주요 키워드

| 키워드 | 설명 | 생성되는 SQL (예시) |
| :--- | :--- | :--- |
| `And` | 여러 조건을 `AND`로 연결합니다. | `findByCityAndMajor(city, major)` |
| `Or` | 여러 조건을 `OR`로 연결합니다. | `findByCityOrMajor(city, major)` |
| `Is`, `Equals` | 동일 조건 (`=`)을 의미합니다. (생략 가능) | `findByName` / `findByNameIs` |
| `Containing` | 문자열 포함 조건 (`LIKE '%...%'`) | `findByMajorContaining("공학")` |
| `StartingWith` | 문자열 시작 조건 (`LIKE '...%'`) | `findByMajorStartingWith("컴퓨터")` |
| `EndingWith` | 문자열 끝 조건 (`LIKE '%...'`) | `findByMajorEndingWith("공학")` |
| `LessThan` | 보다 작음 (`<`) | `findByAgeLessThan(20)` |
| `LessThanEqual` | 보다 작거나 같음 (`<=`) | `findByAgeLessThanEqual(20)` |
| `GreaterThan` | 보다 큼 (`>`) | `findByAgeGreaterThan(20)` |
| `GreaterThanEqual` | 보다 크거나 같음 (`>=`) | `findByAgeGreaterThanEqual(20)` |
| `OrderBy` | 결과를 정렬합니다. | `findAllByOrderByAgeDesc()` |
