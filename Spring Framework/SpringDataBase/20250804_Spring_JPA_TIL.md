
# QueryDSL : 서브쿼리와 동적 쿼리

## 1\. QueryDSL 서브쿼리 (Subquery)

QueryDSL을 사용하면 JPA에서 직접 지원하지 않는 `FROM` 절의 서브쿼리를 제외한 `SELECT`와 `WHERE` 절의 서브쿼리를 구현할 수 있습니다. 서브쿼리는 `com.querydsl.jpa.JPAExpressions` 클래스를 사용하여 생성합니다.

### 1.1. 주요 개념 및 사용법

  * **`JPAExpressions`**: 서브쿼리를 생성하기 위한 유틸리티 클래스입니다. `select`, `selectFrom` 등의 메서드를 제공합니다.
  * **Q-Type 별칭(alias)**: 같은 엔티티를 서브쿼리에서 함께 사용해야 할 경우, 별칭을 지정하여 구분해야 합니다. `QAlbum subAlbum = new QAlbum("subAlbum");`과 같이 새로운 Q-Type 인스턴스를 생성하여 사용합니다.
  * **결과 반환**:
      * `.where(필드.in(subquery))`: 서브쿼리 결과에 포함되는 경우
      * `.where(필드.notIn(subquery))`: 서브쿼리 결과에 포함되지 않는 경우
      * `.where(subquery.exists())`: 서브쿼리 결과가 존재하는 경우
      * `.where(subquery.notExists())`: 서브쿼리 결과가 존재하지 않는 경우
      * `.where(필드.gt(subquery))`: 스칼라 서브쿼리 결과보다 큰 경우 (gt, goe, lt, loe, eq 등 비교 연산자 사용 가능)

### 1.2. 서브쿼리 예제 코드 (`QueryDslSubqueryTest.java`)

#### 📝 테스트 환경 설정

테스트를 위해 `@BeforeEach`를 사용하여 그룹, 아이돌, 앨범 데이터를 미리 저장합니다.

```java
@SpringBootTest
@Transactional
@Rollback(false)
public class QueryDslSubqueryTest {

    @Autowired
    IdolRepository idolRepository;
    @Autowired
    GroupRepository groupRepository;
    @Autowired
    AlbumRepository albumRepository;
    @Autowired
    JPAQueryFactory factory;

    @BeforeEach
    void setUp() {
        //given
        Group leSserafim = new Group("르세라핌");
        Group ive = new Group("아이브");
        Group bts = new Group("방탄소년단");
        Group newjeans = new Group("뉴진스");

        groupRepository.save(leSserafim);
        groupRepository.save(ive);
        groupRepository.save(bts);
        groupRepository.save(newjeans);

        // ... (아이돌, 앨범 데이터 저장)
    }
    // ... 테스트 메서드들
}
```

-----

#### 📌 예제 1: 스칼라 서브쿼리 - 특정 그룹의 평균 나이보다 많은 아이돌 조회

`WHERE` 절에서 단일 값을 반환하는 서브쿼리(스칼라 서브쿼리)를 사용하여 조건을 설정합니다.

```java
@Test
@DisplayName("특정 아이돌 그룹의 평균 나이보다 많은 아이돌 정보 조회")
void subqueryTest1 () {
    //given
    String groupName = "르세라핌";
    //when
    List<Idol> idolList = factory
            .selectFrom(idol)
            .where(idol.age.gt( // gt: greater than
                    // 서브쿼리 시작
                    JPAExpressions
                            .select(idol.age.avg()) // 르세라핌의 평균 나이 조회
                            .from(idol)
                            .where(idol.group.groupName.eq(groupName))
            ))
            .fetch();
    //then
    idolList.forEach(System.out::println);
}
```

  * **핵심 로직**: `JPAExpressions.select(idol.age.avg())`를 통해 '르세라핌' 그룹의 평균 나이를 구하고, `where(idol.age.gt(...))`를 사용하여 해당 평균 나이보다 나이가 많은 모든 아이돌을 조회합니다.

-----

#### 📌 예제 2: `IN` 절 서브쿼리 - 그룹별 가장 최근 발매 앨범 조회

상관관계 서브쿼리(Correlated Subquery)를 사용하여 메인 쿼리의 데이터를 서브쿼리에서 참조하는 복잡한 케이스입니다.

  * **참고 SQL**:

    ```sql
    SELECT G.group_name, A.album_name, A.release_year
    FROM tbl_group G
    INNER JOIN tbl_album A
    ON G.group_id = A.group_id
    WHERE A.album_id IN (
        SELECT S.album_id
        FROM tbl_album S
        WHERE S.group_id = A.group_id -- 상관관계
            AND S.release_year = (
                SELECT MAX(release_year)
                FROM tbl_album
                WHERE S.group_id = A.group_id -- 상관관계
            )
    )
    ```

  * **QueryDSL 코드**:

    ```java
    @Test
    @DisplayName("그룹별로 가장 최근에 발매된 앨범 정보 조회")
    void subqueryTest2() {
        //given
        // 서브쿼리에서 album 엔티티를 구분하기 위해 새로운 Q-Type 인스턴스 생성
        QAlbum albumA = new QAlbum("albumA"); // 메인 쿼리용
        QAlbum albumS = new QAlbum("albumS"); // 서브 쿼리용

        //when
        List<Tuple> idolTuples = factory
                .select(group.groupName, albumA.albumName, albumA.releaseYear)
                .from(group)
                .innerJoin(group.albums, albumA) // 메인 쿼리의 앨범
                .where(albumA.id.in(
                        JPAExpressions
                                .select(albumS.id)
                                .from(albumS)
                                .where(
                                        // 메인 쿼리의 그룹 ID와 서브쿼리의 그룹 ID가 같은지 비교 (상관관계)
                                        albumS.group.id.eq(albumA.group.id)
                                                .and(
                                                        // 그룹 내에서 가장 최신 발매년도인지 확인하는 서브-서브쿼리
                                                        albumS.releaseYear.eq(
                                                                JPAExpressions
                                                                        .select(albumS.releaseYear.max())
                                                                        .from(albumS)
                                                                        .where(albumS.group.id.eq(albumA.group.id)) // 상관관계
                                                        )
                                                )
                                )
                ))
                .fetch();

        //then
        for (Tuple tuple : idolTuples) {
            String groupName = tuple.get(group.groupName);
            String albumName = tuple.get(albumA.albumName);
            Integer releaseYear = tuple.get(albumA.releaseYear);

            System.out.println("\nGroup: " + groupName
                    + ", Album: " + albumName
                    + ", Release Year: " + releaseYear);
        }
    }
    ```

-----

#### 📌 예제 3: `HAVING` 절을 사용한 서브쿼리 - 특정 연도에 2개 이상 앨범 발매 그룹 조회

`groupBy`와 `having`을 사용해 서브쿼리에서 조건을 만족하는 그룹 ID 목록을 찾고, 메인 쿼리에서 해당 그룹들을 조회합니다.

```java
@Test
@DisplayName("특정 연도에 발매된 앨범 수가 2개 이상인 그룹 조회")
void testFindGroupsWithMultipleAlbumsInYear() {

    int targetYear = 2022;

    // 서브쿼리에서 사용할 Q-Type
    QAlbum subAlbum = new QAlbum("subAlbum");

    // 서브쿼리: 각 그룹별로 특정 연도에 발매된 앨범 수를 계산
    JPQLQuery<Long> subQuery = JPAExpressions
            .select(subAlbum.group.id)
            .from(subAlbum)
            .where(subAlbum.releaseYear.eq(targetYear))
            .groupBy(subAlbum.group.id)
            .having(subAlbum.count().goe(2L)); // 2개 이상

    // 메인쿼리: 서브쿼리의 결과(그룹 ID)와 일치하는 그룹 조회
    List<Group> result = factory
            .selectFrom(group)
            .where(group.id.in(subQuery))
            .fetch();

    assertFalse(result.isEmpty());
    for (Group g : result) {
        System.out.println("\nGroup: " + g.getGroupName());
    }
}
```

-----

#### 📌 예제 4: `NOT EXISTS` - 그룹 없는 아이돌 조회

`EXISTS` 또는 `NOT EXISTS`를 사용하여 서브쿼리의 결과 존재 유무로 메인 쿼리의 결과를 필터링합니다.

```java
@Test
@DisplayName("그룹이 존재하지 않는 아이돌 조회")
void testFindIdolsWithoutGroup() {

    // 서브쿼리: 아이돌이 특정 그룹에 속하는지 확인 (이 경우, idol.group.id가 null인지 아닌지 판별)
    JPQLQuery<Long> subQuery = JPAExpressions
            .select(group.id)
            .from(group)
            .where(group.id.eq(idol.group.id)); // idol.group.id가 null이면 이 조건은 false가 됨

    // 메인쿼리: 서브쿼리 결과가 존재하지 않는(연관된 그룹이 없는) 아이돌 조회
    List<Idol> result = factory
            .selectFrom(idol)
            .where(subQuery.notExists())
            .fetch();

    assertFalse(result.isEmpty());
    for (Idol i : result) {
        System.out.println("\nIdol: " + i.getIdolName());
    }
}
```

  * **실행 원리**: `idol` 테이블의 각 행을 순회하면서 `subQuery`를 실행합니다. `idol.group.id`가 `null`이면 `group.id.eq(null)`은 참이 되지 않아 서브쿼리 결과가 없고, `notExists()` 조건이 참이 되어 해당 아이돌이 최종 결과에 포함됩니다.

-----

#### 📌 예제 5: `NOT EXISTS` - 특정 연도에 앨범을 내지 않은 그룹 조회

```java
@Test
@DisplayName("특정 연도에 발매된 앨범이 없는 그룹 조회")
void testFindGroupsWithoutAlbumsInYear() {
    int targetYear = 2023;

    // 서브쿼리: 특정 연도에 앨범을 발매했으며, 그 앨범의 그룹 ID가 메인 쿼리의 그룹 ID와 같은 경우를 찾음
    JPQLQuery<Long> subQuery = JPAExpressions
            .select(album.group.id)
            .from(album)
            .where(album.releaseYear.eq(targetYear)
                    .and(album.group.id.eq(group.id))); // 상관관계 조건

    // 메인쿼리: 위 서브쿼리 결과가 존재하지 않는 그룹을 조회
    List<Group> result = factory
            .selectFrom(group)
            .where(subQuery.notExists())
            .fetch();

    assertFalse(result.isEmpty());
    for (Group g : result) {
        System.out.println("Group: " + g.getGroupName());
    }
}
```

  * `notIn` 대신 `notExists`를 사용하면 서브쿼리가 `null`을 반환하는 경우에도 의도치 않은 결과(전체 결과가 비는 현상)를 방지할 수 있어 더 안전합니다.

-----

## 2\. QueryDSL 동적 쿼리 (Dynamic Query)

동적 쿼리는 애플리케이션 실행 시점에 사용자의 입력이나 특정 조건에 따라 `WHERE` 절이나 `ORDER BY` 절이 달라지는 쿼리를 의미합니다. QueryDSL에서는 `BooleanBuilder`와 `OrderSpecifier`를 사용하여 동적 쿼리를 쉽게 구현할 수 있습니다.

### 2.1. 주요 개념 및 사용법

  * **`BooleanBuilder`**: `WHERE` 절의 조건을 동적으로 조합하기 위한 클래스입니다. `and()`와 `or()` 메서드를 사용하여 조건을 연결합니다.
  * **`OrderSpecifier`**: `ORDER BY` 절의 정렬 조건을 동적으로 생성합니다. Q-Type 필드에 `.asc()` (오름차순) 또는 `.desc()` (내림차순)를 붙여 생성합니다.

### 2.2. 동적 쿼리 예제 코드 (`QueryDslDynamicTest.java`)

#### 📌 예제 1: `BooleanBuilder`를 사용한 동적 조건 검색

검색 조건(이름, 성별, 나이 범위 등)이 `null`이 아닐 경우에만 `WHERE` 절에 해당 조건을 추가합니다.

```java
@Test
@DisplayName("동적 쿼리를 사용한 간단한 아이돌 조회")
void dynamicTest1() {
    //given
    String name = "김채원"; name = null; // 조건 활성화/비활성화 테스트
    String genderParam = "여"; //genderParam = null;
    Integer minAge = 20;
    Integer maxAge = 25; maxAge = null;

    // 동적 쿼리를 위한 BooleanBuilder 생성
    BooleanBuilder booleanBuilder = new BooleanBuilder();

    // 각 파라미터가 null이 아니면 조건 추가
    if (name != null) {
        booleanBuilder.and(idol.idolName.eq(name));
    }
    if (genderParam != null) {
        booleanBuilder.and(idol.gender.eq(genderParam));
    }
    if (minAge != null) {
        booleanBuilder.and(idol.age.goe(minAge)); // goe: greater or equal
    }
    if (maxAge != null) {
        booleanBuilder.and(idol.age.loe(maxAge)); // loe: less or equal
    }

    //when
    List<Idol> result = factory
            .selectFrom(idol)
            .where(booleanBuilder) // .where()에 BooleanBuilder 객체를 전달
            .fetch();

    //then
    assertFalse(result.isEmpty());
    for (Idol i : result) {
        System.out.println("\nIdol: " + i.getIdolName() + ", Gender: " + i.getGender());
    }
}
```

  * `BooleanBuilder`는 `where()` 메서드에 직접 전달될 수 있습니다. 만약 빌더에 아무런 조건도 추가되지 않으면 `where(null)`과 동일하게 동작하여 `WHERE` 절이 없는 쿼리가 실행됩니다.

-----

#### 📌 예제 2: `OrderSpecifier`를 사용한 동적 정렬

정렬 기준(`sortBy`)과 순서(`ascending`) 파라미터에 따라 `ORDER BY` 절을 동적으로 변경합니다.

```java
@Test
@DisplayName("동적 정렬을 사용한 아이돌 조회")
void dynamicTest2() {
    //given
    String sortBy = "age"; // 정렬 기준: age, idolName, groupName
    boolean ascending = true; // 정렬 순서: true(오름차순), false(내림차순)

    //when
    // 정렬 조건을 담을 OrderSpecifier 변수 선언
    OrderSpecifier<?> specifier = null;

    // 동적 정렬 조건 생성
    switch (sortBy) {
        case "age":
            specifier = ascending ? idol.age.asc() : idol.age.desc();
            break;
        case "idolName":
            specifier = ascending ? idol.idolName.asc() : idol.idolName.desc();
            break;
        case "groupName":
            // null 값을 가진 그룹명을 뒤로 보내려면 .nullsLast() 추가
            specifier = ascending ? idol.group.groupName.asc().nullsLast() : idol.group.groupName.desc().nullsLast();
            break;
    }

    List<Idol> result = factory
            .selectFrom(idol)
            .orderBy(specifier) // .orderBy()에 OrderSpecifier 객체를 전달
            .fetch();

    //then
    assertFalse(result.isEmpty());
    for (Idol i : result) {
        System.out.println("\nIdol: " + i.getIdolName()
                + ", Gender: " + i.getGender()
                + ", age: " + i.getAge()
        );
    }
}
```

  * `switch` 문을 사용하여 `sortBy` 문자열 값에 따라 다른 `OrderSpecifier`를 생성하고, 이를 `orderBy()` 메서드에 전달하여 동적 정렬을 구현합니다.

-----

## 3\. 사용자 정의 리포지토리와 DTO 프로젝션

QueryDSL을 실제 애플리케이션에 적용할 때는 Spring Data JPA의 리포지토리와 통합하여 사용합니다. 복잡한 조회 결과는 DTO(Data Transfer Object)로 직접 매핑하여 반환하는 것이 효율적입니다.

### 3.1. 구현 흐름

1.  **`GroupRepositoryCustom` (Interface)**: QueryDSL로 구현할 메서드를 선언하는 인터페이스를 작성합니다.
2.  **`GroupRepositoryImpl` (Implementation)**: `GroupRepositoryCustom` 인터페이스를 구현합니다. 클래스명은 반드시 `(리포지토리명)Impl`로 끝나야 Spring Data JPA가 인식합니다. `JPAQueryFactory`를 주입받아 쿼리를 작성합니다.
3.  **`GroupRepository` (Main Repository)**: 기존 Spring Data JPA 리포지토리 인터페이스가 `GroupRepositoryCustom`을 상속받도록 합니다.
4.  **DTO (`GroupAverageAge`)**: 조회 결과를 담을 DTO 클래스를 정의합니다. QueryDSL의 `Projections`를 사용하여 결과를 DTO로 바로 매핑할 수 있습니다.

### 3.2. 관련 코드

#### 📄 `GroupRepositoryCustom.java` - 사용자 정의 메서드 선언

```java
package com.spring.database.querydsl.repository;

import com.spring.database.querydsl.dto.GroupAverageAge;
import java.util.List;

// queryDSL이나 다른 ORM라이브러리를 JPA와 결합하기 위한 추가 인터페이스
public interface GroupRepositoryCustom {
    // 그룹별 평균 나이 조회
    List<GroupAverageAge> groupAverage();
}
```

#### 📄 `GroupRepositoryImpl.java` - QueryDSL 구현체

```java
package com.spring.database.querydsl.repository;

import com.querydsl.core.types.Projections;
import com.querydsl.jpa.impl.JPAQueryFactory;
import com.spring.database.querydsl.dto.GroupAverageAge;
import lombok.RequiredArgsConstructor;
import java.util.List;
import static com.spring.database.querydsl.entity.QIdol.*;

@RequiredArgsConstructor
public class GroupRepositoryImpl implements GroupRepositoryCustom {

    // queryDsl을 사용하기 위한 의존객체
    private final JPAQueryFactory factory;

    @Override
    public List<GroupAverageAge> groupAverage() {
        return factory
                .select(
                        // Projections를 사용하여 조회 결과를 DTO 생성자로 매핑
                        Projections.constructor(
                                GroupAverageAge.class, // 사용할 DTO 클래스
                                idol.group.groupName,  // 생성자 파라미터 1
                                idol.age.avg()         // 생성자 파라미터 2
                        )
                )
                .from(idol)
                .groupBy(idol.group)
                .fetch();
    }
}
```

  * **`Projections.constructor(DTO.class, ...)`**: 조회 결과를 지정된 DTO 클래스의 생성자를 통해 객체로 만듭니다. `select` 절의 순서와 DTO 생성자의 파라미터 순서 및 타입이 일치해야 합니다.

#### 📄 `GroupAverageAge.java` - 결과 매핑 DTO

```java
package com.spring.database.querydsl.dto;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.*;

@Getter @Setter @ToString @EqualsAndHashCode
@NoArgsConstructor @AllArgsConstructor @Builder
// 그룹명과 평균나이를 매핑할 클래스
public class GroupAverageAge {

    @JsonProperty("name")
    private String groupName;
    @JsonProperty("age")
    private Double averageAge;

    // QueryDSL의 Projections.constructor()가 이 생성자를 사용합니다.
    // 매개변수 순서와 타입이 select 절과 일치해야 합니다.
}
```

#### 📄 `IdolService.java` & `IdolController.java` - 서비스 및 컨트롤러 계층

서비스는 리포지토리를 호출하고, 컨트롤러는 서비스의 결과를 API 응답으로 반환합니다.

```java
// IdolService.java
@Service
@RequiredArgsConstructor
public class IdolService {
    private final GroupRepository groupRepository;

    // 그룹별 평균나이를 조회
    public List<GroupAverageAge> average() {
        // JpaRepository가 상속받은 사용자 정의 메서드 호출
        return groupRepository.groupAverage();
    }
}

// IdolController.java
@RestController
@RequestMapping("/api/query/idols")
@RequiredArgsConstructor
public class IdolController {
    private final IdolService idolService;

    @GetMapping("/avg")
    public ResponseEntity<?> avg() {
        return ResponseEntity.ok().body(idolService.average());
    }
}
```
