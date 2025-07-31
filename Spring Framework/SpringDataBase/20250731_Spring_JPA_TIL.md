## **1. QueryDSL 기본 개념 및 설정**

### **1.1. QueryDSL이란?**

QueryDSL은 정적 타입을 사용하여 SQL 및 JPQL 쿼리를 생성할 수 있게 해주는 자바 프레임워크입니다. 문자열 기반의 쿼리(JPQL, Native SQL)와 달리, 컴파일 시점에 문법 오류를 발견할 수 있어 안정성이 높고, IDE의 자동 완성 기능의 도움을 받을 수 있어 생산성을 향상시킵니다.

  - **JPQL/SQL의 단점**: 문자열 기반이라 실행 시점에 오류를 발견하게 되며, 오타나 문법 오류에 취약합니다.
  - **QueryDSL의 장점**:
      - **컴파일 시점 오류 발견**: 쿼리 문법 오류를 컴파일 단계에서 잡아낼 수 있습니다.
      - **IDE 지원**: 코드 자동 완성을 통해 개발 생산성이 향상됩니다.
      - **동적 쿼리 작성 용이**: 복잡한 조건의 동적 쿼리를 직관적이고 간결하게 작성할 수 있습니다.
      - **자바 코드로 쿼리 작성**: SQL과 유사한 문법을 자바 코드로 작성하여 가독성이 좋습니다.

> **쿼리 선택 가이드라인**
>
>   - **간단한 쿼리 (CRUD, 단순 조회)**: Spring Data JPA
>   - **조금 복잡한 쿼리 (간단한 조인, 그룹핑)**: QueryDSL
>   - **매우 복잡한 쿼리 or DBMS 종속적 쿼리**: 순수 SQL (JDBC or Native Query)

-----

### **1.2. QueryDSL 설정**

QueryDSL을 사용하기 위해서는 `JPAQueryFactory` 객체를 스프링 컨테이너에 빈(Bean)으로 등록해야 합니다. `JPAQueryFactory`는 QueryDSL 쿼리를 생성하고 실행하는 핵심적인 역할을 합니다.

#### **`QueryDslConfig.java`**

`JPAQueryFactory`를 스프링 빈으로 수동 등록하는 설정 파일입니다.

```java
package com.spring.database.querydsl.config;

import com.querydsl.jpa.impl.JPAQueryFactory;
import jakarta.persistence.EntityManager;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

// QueryDsl의 핵심객체 JPAQueryFactory를 스프링에게 맡기는 설정(빈 수동 등록)
@Configuration
@RequiredArgsConstructor
public class QueryDslConfig {

    private final EntityManager em;

    @Bean
    public JPAQueryFactory factory(){
        return new JPAQueryFactory(em);
    }

}
```

  - `@Configuration`: 이 클래스가 스프링의 설정 파일임을 나타냅니다.
  - `@RequiredArgsConstructor`: `final` 필드에 대한 생성자를 자동으로 생성합니다.
  - `@Bean`: `factory()` 메서드가 반환하는 `JPAQueryFactory` 객체를 스프링 빈으로 등록합니다. 이렇게 등록된 빈은 다른 컴포넌트에서 `@Autowired`를 통해 주입받아 사용할 수 있습니다.
  - **`EntityManager` (em)**: JPA의 모든 영속성 작업을 처리하는 핵심 객체입니다. `JPAQueryFactory`는 이 `EntityManager`를 통해 데이터베이스와 상호작용합니다.

-----

### **1.3. 엔티티 및 리포지토리**

QueryDSL 쿼리를 작성하기 위해 먼저 JPA 엔티티와 리포지토리를 정의해야 합니다.

#### **엔티티(Entity)**

##### **`Idol.java`**

```java
package com.spring.database.querydsl.entity;

import jakarta.persistence.*;
import lombok.*;


@Getter
@Setter
@ToString(exclude = {"group"})
@EqualsAndHashCode
@NoArgsConstructor
@AllArgsConstructor
@Builder

@Entity
@Table(name = "tbl_idol")
public class Idol {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "idol_id")
    private Long id;

    private String idolName;

    private int age;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "group_id")
    private Group group;

    public Idol(String idolName, int age, Group group) {
        this.idolName = idolName;
        this.age = age;
        this.group = group;
    }

    // 아이돌의 그룹을 변경하는 메서드
    public void changeGroup(Group group){
        this.group = group;
        group.getIdols().add(this);
    }
}
```

##### **`Group.java`**

```java
package com.spring.database.querydsl.entity;

import jakarta.persistence.*;
import lombok.*;

import java.util.ArrayList;
import java.util.List;

@EqualsAndHashCode
@Getter
@Setter
@ToString(exclude = {"idols"})
@AllArgsConstructor
@NoArgsConstructor
@Builder

@Entity
@Table(name = "tbl_group")
public class Group {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "group_id")
    private Long id;

    private String groupName;

    @OneToMany(mappedBy = "group", cascade = CascadeType.ALL, orphanRemoval = true)
    List<Idol> idols = new ArrayList<>();

    public Group(String groupName) {
        this.groupName = groupName;
    }

    // 양방향 리스트 편의 메서드
    // 데이터 추가
    public void addIdol(Idol idol){
        this.idols.add(idol);
        idol.setGroup(this);
    }

    // 데이터 제거
    public void removeIdol(Idol idol){
        this.idols.remove(idol);
        idol.setGroup(null);
    }
}
```

| 어노테이션 | 설명 |
| :--- | :--- |
| `@Entity` | 해당 클래스가 JPA 엔티티임을 나타냅니다. |
| `@Table` | 엔티티와 매핑할 테이블을 지정합니다. |
| `@Id` | 테이블의 기본 키(PK) 필드임을 나타냅니다. |
| `@GeneratedValue` | 기본 키 생성 전략을 정의합니다. (e.g., `IDENTITY`, `SEQUENCE`) |
| `@Column` | 객체 필드를 테이블의 컬럼에 매핑합니다. |
| `@ManyToOne` | 다대일(N:1) 관계를 정의합니다. `Idol`이 `Group`에 속합니다. |
| `@OneToMany` | 일대다(1:N) 관계를 정의합니다. `Group`은 여러 `Idol`을 가집니다. |
| `fetch = FetchType.LAZY` | 지연 로딩. 연관된 엔티티를 실제 사용할 때 조회하여 성능을 최적화합니다. |
| `mappedBy = "group"` | 연관관계의 주인이 아님을 명시합니다. (`Idol` 엔티티의 `group` 필드가 주인입니다.) |
| `cascade = CascadeType.ALL` | 영속성 전이. 부모 엔티티(`Group`)의 상태 변화가 자식 엔티티(`Idol`)에 전파됩니다. |
| `orphanRemoval = true` | 고아 객체 제거. 부모 엔티티와의 연관관계가 끊어진 자식 엔티티를 자동으로 삭제합니다. |

#### **리포지토리(Repository)**

Spring Data JPA를 사용하여 기본적인 CRUD 기능을 제공하는 인터페이스입니다.

##### **`IdolRepository.java`**

```java
package com.spring.database.querydsl.repository;

import com.spring.database.querydsl.entity.Idol;
import org.springframework.data.jpa.repository.JpaRepository;

public interface IdolRepository extends JpaRepository<Idol,Long> {
}
```

##### **`GroupRepository.java`**

```java
package com.spring.database.querydsl.repository;

import com.spring.database.querydsl.entity.Group;
import org.springframework.data.jpa.repository.JpaRepository;

public interface GroupRepository extends JpaRepository<Group, Long> {
}
```

-----

## **2. QueryDSL 기본 문법 및 테스트**

`QueryDslBasicTest.java` 파일을 통해 다양한 조회 방법을 비교하고 QueryDSL의 기본 문법을 학습합니다.

### **2.1. 테스트 환경 설정 (`@BeforeEach`)**

모든 테스트 전에 실행될 초기 데이터를 설정합니다.

```java
@SpringBootTest
@Transactional
class QueryDslBasicTest {

    @Autowired
    IdolRepository idolRepository;

    @Autowired
    GroupRepository groupRepository;

    @Autowired
    EntityManager em;  // 순수 JPA의 핵심객체

    @Autowired
    JdbcTemplate jdbcTemplate;  // JDBC의 핵심객체

    @Autowired
    JPAQueryFactory factory; // QueryDsl 핵심객체 (스프링 자동관리를 안하기 때문에 @Bean으로 수동 등록)

    @BeforeEach
    void setUp() {
        //given
        Group leSserafim = new Group("르세라핌");
        Group ive = new Group("아이브");

        groupRepository.save(leSserafim);
        groupRepository.save(ive);

        Idol idol1 = new Idol("김채원", 24, leSserafim);
        Idol idol2 = new Idol("사쿠라", 26, leSserafim);
        Idol idol3 = new Idol("가을", 22, ive);
        Idol idol4 = new Idol("리즈", 20, ive);

        idolRepository.save(idol1);
        idolRepository.save(idol2);
        idolRepository.save(idol3);
        idolRepository.save(idol4);

        em.flush();
        em.clear();
    }
    // ... 테스트 코드 ...
}
```

  - `@Transactional`: 테스트 완료 후 롤백을 수행하여 테스트가 데이터베이스 상태에 영향을 주지 않도록 합니다.
  - `em.flush()`: 영속성 컨텍스트의 변경 내용을 데이터베이스에 동기화합니다.
  - `em.clear()`: 영속성 컨텍스트를 비워 1차 캐시를 초기화합니다.

### **2.2. 다양한 조회 방법 비교**

#### **1) 순수 JPQL**

  - 엔티티 객체를 대상으로 하는 객체지향 쿼리입니다.
  - SQL과 유사하지만, 테이블 대신 엔티티 이름을 사용합니다.
  - 문자열 기반이라 컴파일 시점에 오류를 잡을 수 없습니다.

<!-- end list -->

```java
@Test
@DisplayName("순수 JPQL로 특정 이름의 아이돌 조회하기")
void rawJpqlTest() {
    //given
    String jpql = """
            SELECT i
            FROM Idol i
            WHERE i.idolName = ?1
            """;

    //when
    Idol foundIdol = em.createQuery(jpql, Idol.class)
            .setParameter(1, "가을")
            .getSingleResult();

    //then
    System.out.println("\n\nfoundIdol = " + foundIdol);
    System.out.println("foundIdol.getGroup() = " + foundIdol.getGroup());
}
```

#### **2) 순수 SQL (Native Query)**

  - 데이터베이스 테이블을 직접 대상으로 하는 표준 SQL 쿼리입니다.
  - 특정 DBMS에 종속적인 기능을 사용할 때 유용합니다.
  - JPQL과 마찬가지로 문자열 기반입니다.

<!-- end list -->

```java
@Test
@DisplayName("순수 SQL로 특정 이름의 아이돌 조회하기")
void nativeSqlTest() {
    //given
    String sql = """
            SELECT *
            FROM tbl_idol
            WHERE idol_name = ?1
            """;
    //when
    Idol foundIdol = (Idol) em.createNativeQuery(sql, Idol.class)
            .setParameter(1, "리즈")
            .getSingleResult();

    //then
    System.out.println("\n\nfoundIdol = " + foundIdol);
    System.out.println("foundIdol.getGroup() = " + foundIdol.getGroup());
}
```

#### **3) JDBC**

  - 자바에서 데이터베이스에 연결하고 SQL을 실행하기 위한 표준 API입니다.
  - 코드가 장황하고, 객체-관계 매핑(ORM)을 수동으로 처리해야 합니다.
  - 연관된 엔티티를 조회하려면 추가적인 쿼리가 필요할 수 있습니다.

<!-- end list -->

```java
@Test
@DisplayName("JDBC로 특정이름의 아이돌 조회하기")
void jdbcTest() {
    //given
    String sql = """
            SELECT *
            FROM tbl_idol
            WHERE idol_name = ?
            """;
    //when
    // JDBC의 queryForObject는 RowMapper를 통해 ResultSet을 객체로 매핑해야 합니다.
    Idol foundIdol = jdbcTemplate.queryForObject(sql,
            (rs, n) -> new Idol(
                    rs.getLong("idol_id"),
                    rs.getString("idol_name"),
                    rs.getInt("age"),
                    null // 그룹 정보는 이 쿼리만으로 가져올 수 없음
            ),
            "김채원"
    );

    // 그룹 정보를 가져오기 위해 별도의 쿼리 실행
    Group group = jdbcTemplate.queryForObject("""
                    SELECT * FROM tbl_group WHERE group_id = ?
                    """,
            (rs, n) -> new Group(
                    rs.getLong("group_id"),
                    rs.getString("group_name"),
                    null
            ),
            1
    );

    foundIdol.setGroup(group);

    //then
    System.out.println("\n\nfoundIdol = " + foundIdol);
    System.out.println("foundIdol.getGroup() = " + foundIdol.getGroup());
}
```

### **2.3. QueryDSL 기본 조회**

QueryDSL은 빌드 과정에서 엔티티를 기반으로 `Q-Type` 클래스(예: `QIdol.java`)를 생성합니다. 이 `Q-Type` 객체를 사용하여 쿼리를 타입-세이프하게 작성합니다.

```java
// QIdol.java의 idol을 static import하여 사용
import static com.spring.database.querydsl.entity.QIdol.*;

// ...

@Test
@DisplayName("QueryDsl로 특정 이름의 아이돌 조회하기")
void queryDslTest () {
    //given
    // JPAQueryFactory factory = new JPAQueryFactory(em); // @Bean 등록으로 생략 가능

    //when
    Idol foundIdol = factory
            .selectFrom(idol) // Q-Type 객체(static import)
            .where(idol.idolName.eq("사쿠라"))
            .fetchOne(); // 단일 행 조회
    //then
    System.out.println("\n\nfoundIdol = " + foundIdol);
    System.out.println("foundIdol.getGroup() = " + foundIdol.getGroup());
}
```

  - `factory.selectFrom(idol)`: `idol` 테이블(엔티티)에서 조회를 시작합니다.
  - `where(idol.idolName.eq("사쿠라"))`: `idolName`이 "사쿠라"와 같은 조건을 추가합니다.
  - `fetchOne()`: 조건에 맞는 결과를 단일 객체로 반환합니다. 결과가 없거나 둘 이상이면 예외가 발생할 수 있습니다.

### **2.4. 검색 조건절 (Where Clause)**

`where()` 메서드에 다양한 조건을 조합하여 사용할 수 있습니다.

```java
@Test
@DisplayName("이름 AND 나이로 아이돌 조회하기")
void searchTest() {
    //given
    String name = "리즈";
    int age = 20;

    //when
    Idol foundIdol = factory
            .selectFrom(idol)
            .where(
                    idol.idolName.eq(name)
                            .and(idol.age.eq(age)) // AND 조건
            )
            .fetchOne();
            
    // and 대신 쉼표(,)를 사용해도 동일하게 AND 조건으로 처리됩니다.
    // .where(idol.idolName.eq(name), idol.age.eq(age))

    //then
    System.out.println("\n\nfoundIdol = " + foundIdol);
    System.out.println("foundIdol.getGroup() = " + foundIdol.getGroup());
}
```

#### **주요 조건 메서드**

| 메서드 | JPQL/SQL | 설명 |
| :--- | :--- | :--- |
| `.eq(value)` | `=` | 동등 비교 |
| `.ne(value)` / `.not()` | `!=` or `<>` | 부등 비교 |
| `.in(val1, val2, ...)` | `IN` | 목록 포함 여부 |
| `.notIn(val1, val2, ...)`| `NOT IN` | 목록 미포함 여부 |
| `.between(start, end)` | `BETWEEN` | 범위 비교 |
| `.gt(value)` | `>` | 보다 큼 (Greater Than) |
| `.goe(value)` | `>=` | 크거나 같음 (Greater Or Equal) |
| `.lt(value)` | `<` | 보다 작음 (Less Than) |
| `.loe(value)` | `<=` | 작거나 같음 (Less Or Equal) |
| `.isNotNull()` | `IS NOT NULL`| Null이 아닌지 검사 |
| `.like(pattern)` | `LIKE` | 문자열 패턴 매칭 (`%`, `_` 사용) |
| `.contains(str)` | `LIKE '%str%'` | 문자열 포함 여부 |
| `.startsWith(str)` | `LIKE 'str%'` | 문자열 시작 여부 |
| `.endsWith(str)` | `LIKE '%str'` | 문자열 종료 여부 |

### **2.5. 다양한 조회 결과 반환 (Fetch)**

쿼리 결과를 다양한 형태로 반환받을 수 있습니다.

```java
@Test
@DisplayName("다양한 조회 결과 반환하기")
void fetchTest() {
    //fetch() : 다중 행 조회
    List<Idol> idolList = factory
            .selectFrom(idol)
            .fetch();

    System.out.println("\n\nfetch = ================================");
    idolList.forEach(System.out::println);

    // fetchFirst() : 다중행 조회에서 첫번째 행을 반환
    Idol fetchFirst = factory
            .selectFrom(idol)
            .where(idol.age.loe(22))
            .fetchFirst();

    System.out.println("\n\nfetchFirst = ================================");
    System.out.println("fetchFirst = " + fetchFirst);

    // 단일 행 조회 시 NullPointerException(NPE)에 취약하므로 Optional 사용을 권장
    Optional<Idol> fetchOne = Optional.ofNullable(
            factory
                    .selectFrom(idol)
                    .where(idol.idolName.eq("김채원123")) // 존재하지 않는 데이터
                    .fetchOne()
    ) ;

    // 결과가 없으면 기본 Idol 객체 반환
    Idol foundIdol = fetchOne.orElse(new Idol());
}
```

| 메서드 | 설명 |
| :--- | :--- |
| `fetch()` | 조건에 맞는 모든 결과를 `List` 형태로 반환합니다. 결과가 없으면 빈 리스트를 반환합니다. |
| `fetchOne()` | 단일 결과를 반환합니다. 결과가 없으면 `null`, 둘 이상이면 `NonUniqueResultException`을 발생시킵니다. |
| `fetchFirst()` | 조회 결과 중 첫 번째 한 건만 반환합니다. `limit(1).fetchOne()`과 동일합니다. |

### **2.6. 조건절 활용 예제**

`QueryDslBasicTest.java`의 추가 테스트 케이스들입니다.

```java
@Test
@DisplayName("나이가 24세 이상인 아이돌 조회")
void testAgeGoe() {
    // given
    int ageThreshold = 24;

    // when
    List<Idol> result = factory
            .selectFrom(idol)
            .where(idol.age.goe(ageThreshold)) // age >= 24
            .fetch();

    // then
    assertFalse(result.isEmpty());
    for (Idol idol : result) {
        System.out.println("\n\nIdol: " + idol);
        assertTrue(idol.getAge() >= ageThreshold);
    }
}

@Test
@DisplayName("이름에 '김'이 포함된 아이돌 조회")
void testNameContains() {
    // given
    String substring = "김";

    // when
    List<Idol> result = factory
            .selectFrom(idol)
            .where(idol.idolName.contains(substring)) // idolName LIKE '%김%'
            .fetch();

    // then
    assertFalse(result.isEmpty());
    for (Idol idol : result) {
        System.out.println("Idol: " + idol);
        assertTrue(idol.getIdolName().contains(substring));
    }
}

@Test
@DisplayName("나이가 20세에서 25세 사이인 아이돌 조회")
void testAgeBetween() {
    // given
    int ageStart = 20;
    int ageEnd = 25;

    // when
    List<Idol> result = factory
            .selectFrom(idol)
            .where(idol.age.between(ageStart, ageEnd)) // age BETWEEN 20 AND 25
            .fetch();

    // then
    assertFalse(result.isEmpty());
    for (Idol idol : result) {
        System.out.println("Idol: " + idol);
        assertTrue(idol.getAge() >= ageStart && idol.getAge() <= ageEnd);
    }
}

@Test
@DisplayName("르세라핌 그룹에 속한 아이돌 조회")
void testGroupEquals() {
    /*
    -- QueryDSL은 내부적으로 아래와 같은 JOIN 쿼리를 생성합니다.
    SELECT i.*
    FROM tbl_idol i
    INNER JOIN tbl_group g
    ON i.group_id = g.group_id
    WHERE g.group_name = '르세라핌'
    */
    // given
    String groupName = "르세라핌";

    // when
    List<Idol> result = factory
            .selectFrom(idol)
            // 연관관계 필드를 통해 자연스럽게 조인 발생
            .where(idol.group.groupName.eq(groupName))
            .fetch();

    // then
    assertFalse(result.isEmpty());
    for (Idol i : result) {
        System.out.println("Idol: " + i);
        assertEquals(groupName, i.getGroup().getGroupName());
    }
}
```

-----

## **3. QueryDSL 정렬 및 페이징**

`QueryDslSortTest.java` 파일을 통해 정렬 및 페이징 처리 방법을 학습합니다.

### **3.1. 정렬 (OrderBy)**

`orderBy()` 메서드를 사용하여 조회 결과를 정렬합니다.

  - `asc()`: 오름차순 (Ascending)
  - `desc()`: 내림차순 (Descending)

<!-- end list -->

```java
@Test
@DisplayName("기본 정렬 사용법")
void sortingTest() {
    //given

    //when
    List<Idol> idolList = factory
            .selectFrom(idol)
            // ORDER BY age DESC, idol_name DESC
            // 첫 번째 정렬 조건(나이)이 같을 경우, 두 번째 조건(이름)으로 정렬
            .orderBy(idol.age.desc(), idol.idolName.desc())
            .fetch();
            
    //then
    System.out.println("\n\n================================");
    idolList.forEach(System.out::println);
}

@Test
@DisplayName("아이돌을 이름 기준으로 오름차순으로 정렬하여 조회")
void nameSortTest () {
    //given

    //when
    List<Idol> idolList = factory.selectFrom(idol)
            .orderBy(idol.idolName.asc()) // ORDER BY idol_name ASC
            .fetch();

    //then
    System.out.println("\n\n================================");
    idolList.forEach(System.out::println);
}
```

### **3.2. 페이징 (Paging)**

`offset()`과 `limit()` 메서드를 조합하여 페이징을 구현합니다.

  - `offset(long offset)`: 조회를 시작할 행의 위치 (0부터 시작)
  - `limit(long limit)`: 조회할 데이터의 개수

<!-- end list -->

```java
@Test
@DisplayName("페이징 처리하기")
void pagingTest () {
    //given
    int limit = 2; // 한 페이지당 보여줄 데이터 수
    int pageNo = 1; // 요청된 페이지 번호 (클라이언트에서 전달)

    // 조회 시작 위치 계산 (첫 번째 데이터가 0번)
    int offset = (pageNo - 1) * limit;

    //when
    List<Idol> idolList = factory
            .selectFrom(idol)
            .orderBy(idol.age.desc()) // 정렬 조건이 없으면 페이징의 의미가 희석될 수 있음
            .offset(offset)
            .limit(limit)
            .fetch();

    //then
    System.out.println("\n\n================================");
    idolList.forEach(System.out::println);
}
```

### **3.3. 정렬과 페이징 종합 예제**

필터링(`where`), 정렬(`orderBy`), 페이징(`offset`, `limit`)을 함께 사용하는 예제입니다.

```java
@Test
@DisplayName("아이돌을 나이 기준으로 내림차순 정렬하고, 페이지당 3명씩 페이징 처리하여 1번째 페이지의 아이돌을 조회")
void sortAndPagingTest() {
    //given
    int limit = 3;
    int pageNo = 1;
    int offset = (pageNo - 1) * limit;

    //when
    List<Idol> idolList = factory.selectFrom(idol)
            .orderBy(idol.age.desc())
            .offset(offset)
            .limit(limit)
            .fetch();

    //then
    System.out.println("\n\n================================");
    idolList.forEach(System.out::println);
}

@Test
@DisplayName("\"아이브\" 그룹의 아이돌을 이름 기준으로 오름차순 정렬하고, 페이지당 2명씩 페이징 처리하여 첫 번째 페이지의 아이돌을 조회")
void iveSortAndPagingTest () {
    //given
    int limit = 2;
    int pageNo = 1;
    int offset = (pageNo - 1) * limit;
    String groupName = "아이브";

    //when
    List<Idol> idolList = factory.selectFrom(idol)
            .where(idol.group.groupName.eq(groupName)) // 1. 필터링
            .orderBy(idol.idolName.asc())             // 2. 정렬
            .offset(offset)                           // 3. 페이징
            .limit(limit)                             // 4. 페이징
            .fetch();

    //then
    System.out.println("\n\n================================");
    idolList.forEach(System.out::println);
}
```
