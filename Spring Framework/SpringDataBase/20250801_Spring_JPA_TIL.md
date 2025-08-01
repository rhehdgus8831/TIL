
# 1\. QueryDSL Grouping & Aggregation

SQL의 `GROUP BY`, `HAVING` 절과 집계 함수를 QueryDSL에서 사용하는 방법을 학습합니다.

### 📜 `QueryDslGroupingTest.java`

그룹화 및 집계 함수 관련 테스트 코드입니다.

#### 🧪 `@BeforeEach` - 테스트 데이터 설정

각 테스트 전에 실행되어 필요한 데이터를 미리 저장합니다.

```java
@SpringBootTest
@Transactional
@Rollback(value = false)
public class QueryDslGroupingTest {

    @Autowired
    IdolRepository idolRepository;

    @Autowired
    GroupRepository groupRepository;

    @Autowired
    JPAQueryFactory factory;

    @BeforeEach
    void testInsertData() {
        //given
        Group leSserafim = new Group("르세라핌");
        Group ive = new Group("아이브");
        Group bts = new Group("방탄소년단");
        Group newjeans = new Group("newjeans");

        groupRepository.save(leSserafim);
        groupRepository.save(ive);
        groupRepository.save(bts);
        groupRepository.save(newjeans);

        // ... (아이돌 데이터 저장)
    }
    // ... 이하 테스트 코드
}
```

## 1-1. Tuple을 이용한 특정 컬럼 조회

`select` 절에 원하는 컬럼만 명시하여 조회할 수 있습니다. 조회 결과는 `com.querydsl.core.Tuple` 타입으로 반환됩니다.

```java
@Test
@DisplayName("SELECT 절에서 원하는 컬럼만 조회하는 방법")
void tupleTest() {
    //given

    //when
    // idol 엔터티에서 idolName과 age만 조회
    List<Tuple> idolList = factory
            .select(idol.idolName, idol.age)
            .from(idol)
            .fetch();

    for (Tuple tuple : idolList) {
        // Tuple에서 Q-Type을 이용해 데이터 추출
        String name = tuple.get(idol.idolName);
        Integer age = tuple.get(idol.age);
        System.out.printf("이름: %s , 나이: %d세\n", name, age);
    }
    //then
}
```

### 1-2. `groupBy()`와 집계 함수

SQL의 `GROUP BY`와 `COUNT()`, `SUM()`, `AVG()` 등의 집계 함수를 사용할 수 있습니다.

#### 전체 그룹화 (집계 함수만 사용)

`groupBy` 없이 집계 함수를 사용하면 전체 데이터를 대상으로 연산합니다.

```java
@Test
@DisplayName("전체 그룹화하기 - GROUP BY 없이 집계함수를 사용하는 것")
void groupByTest() {
    //given

    //when
    // 모든 아이돌의 나이 합계 조회
    Integer sumAge = factory
            .select(idol.age.sum()) // SUM(age)
            .from(idol)
            .fetchOne(); // 결과가 단일 행일 때 사용
    //then
    System.out.println("sumAge = " + sumAge);
}
```

#### 그룹별 집계

`groupBy()`를 사용하여 특정 컬럼을 기준으로 그룹화하고 집계 함수를 적용합니다.

```java
@Test
@DisplayName("그룹별 인원수 세기")
void groupByCountTest() {
    /* SQL 쿼리
        SELECT G.group_name, COUNT(*)
        FROM tbl_idol I
        INNER JOIN tbl_group G ON I.group_id = G.group_id
        GROUP BY I.group_id;
    */
    //given

    //when
    // 아이돌 그룹별로 인원수 조회
    List<Tuple> idolCounts = factory
            .select(idol.group.groupName, idol.count())
            .from(idol)
            .groupBy(idol.group.id) // 그룹의 ID로 그룹화
            .fetch();

    for (Tuple tuple : idolCounts) {
        String groupName = tuple.get(idol.group.groupName);
        Long count = tuple.get(idol.count());

        System.out.printf("그룹명: %s , 인원수: %d명\n", groupName, count);
    }
    //then
}
```

### 1-3. `having()` - 그룹화 결과 필터링

`groupBy`를 통해 얻은 결과에 조건을 걸어 필터링합니다.

```java
@Test
@DisplayName("아이돌 그룹별로 아이돌의 그룹명과 평균 나이를 조회하세요." +
        " 평균 나이가 20세와 25세 사이인 그룹만 조회합니다.")
void groupByGroupNameAndAvgAgeTest() {
    //given

    //when
    List<Tuple> tuples = factory.select(idol.group.groupName, idol.age.avg())
            .from(idol)
            .groupBy(idol.group.groupName)
            .having(idol.age.avg().between(20, 25)) // 평균 나이가 20~25세인 그룹만
            .fetch();
    //then
    for (Tuple tuple : tuples) {
        String groupName = tuple.get(idol.group.groupName);
        Double age = tuple.get(idol.age.avg());

        System.out.printf("그룹명 : %s , 평균나이 : %s \n", groupName, age);
    }
}
```

### 1-4. 조회 결과를 DTO로 매핑하기

`Tuple` 대신 DTO(Data Transfer Object)를 사용하면 조회 결과를 더 타입-세이프하고 객체지향적으로 다룰 수 있습니다.

#### 📜 `GroupAverageAge.java` - DTO 클래스

그룹명과 평균 나이를 담기 위한 클래스입니다.

```java
package com.spring.database.querydsl.dto;

import com.querydsl.core.Tuple;
import com.spring.database.querydsl.entity.QIdol;
import lombok.*;

@Getter @Setter @ToString @Builder
@NoArgsConstructor @AllArgsConstructor
@EqualsAndHashCode
// 그룹명과 평균나이를 매핑할 클래스
public class GroupAverageAge {

    private String groupName;
    private Double averageAge;

    // Tuple을 전달받아서 DTO로 변환하는 생성자
    public GroupAverageAge(Tuple tuple) {
        this.groupName = tuple.get(QIdol.idol.group.groupName);
        this.averageAge = tuple.get(QIdol.idol.age.avg());
    }

    // 정적 팩토리 메서드 패턴 (이름을 마음대로 설정 가능)
    // Tuple을 전달받아서 DTO로 변환하는 메서드
    public static GroupAverageAge from(Tuple tuple) {
        return GroupAverageAge.builder()
                .groupName(tuple.get(QIdol.idol.group.groupName))
                .averageAge(tuple.get(QIdol.idol.age.avg()))
                .build();
    }
}
```

#### 📜 `GroupAverageRecord.java` - DTO 레코드 (Java 17+)

`record`를 사용하면 `Getter`, `Setter`, `toString` 등의 보일러플레이트 코드를 자동으로 생성해 더 간결합니다.

```java
package com.spring.database.querydsl.dto;

import com.querydsl.core.Tuple;
import com.spring.database.querydsl.entity.QIdol;
import lombok.Builder;

@Builder
// 레코드 : 자동 보일러 플레이트 생성 (getter, setter ...)
// 자바 17에서 사용 가능
public record GroupAverageRecord(
        String groupName,
        Double average
) {
    // ...
}
```

#### 방법 1: `stream()`과 정적 팩토리 메서드 사용

조회한 `Tuple` 리스트를 `stream()`으로 변환한 후, DTO의 정적 팩토리 메서드를 이용해 DTO 객체로 매핑합니다.

```java
@Test
@DisplayName("Tuple 대신 DTO를 사용해서 조회데이터 매핑하기 ver.1")
void groupAvgDtoTest1() {
    // ...
    List<GroupAverageAge> results = factory
            .select(idol.group.groupName, idol.age.avg())
            .from(idol)
            .groupBy(idol.group.groupName)
            .having(idol.age.avg().between(20, 25))
            .fetch()
            .stream() // List<Tuple>을 Stream<Tuple>으로 변환
            .map(GroupAverageAge::from) // Stream<Tuple>을 Stream<GroupAverageAge>으로 변환
            .collect(Collectors.toList()); // Stream을 다시 List로

    // ... (결과 출력)
}
```

#### 방법 2: `Projections` 사용

QueryDSL이 제공하는 `Projections`를 사용하면 조회 결과를 DTO로 직접 매핑할 수 있어 더 간결합니다.

| `Projections` 메서드 | 설명 |
| :--- | :--- |
| `constructor(Class, ...)` | DTO의 생성자를 호출하여 객체를 생성합니다. DTO의 생성자 파라미터 순서와 `select` 절의 조회 컬럼 순서 및 타입이 일치해야 합니다. |
| `bean(Class, ...)` | DTO의 `setter`를 이용해 값을 주입합니다. 기본 생성자가 필요합니다. |
| `fields(Class, ...)` | DTO의 필드에 직접 값을 주입합니다. `setter` 없이도 동작합니다. |

```java
@Test
@DisplayName("Tuple 대신 DTO를 사용해서 조회데이터 매핑하기 ver.2")
void groupAvgDtoTest2() {
    // ...
    List<GroupAverageAge> results = factory
            .select(
                // Projections.constructor를 사용해 DTO로 바로 매핑
                Projections.constructor(
                        GroupAverageAge.class, // 사용할 DTO 클래스
                        idol.group.groupName,  // 생성자 첫 번째 파라미터
                        idol.age.avg()         // 생성자 두 번째 파라미터
                )
            )
            .from(idol)
            .groupBy(idol.group.groupName)
            .having(idol.age.avg().between(20, 25))
            .fetch();
    // ... (결과 출력)
}
```

### 2-5. `CaseBuilder` - 조건부 로직 처리

SQL의 `CASE` 문처럼 복잡한 조건부 로직을 쿼리 내에서 처리할 수 있습니다.

```java
@Test
@DisplayName("연령대별 아이돌 수 조회")
void testAgeGroupBy() {
    // given
    // 연령대를 기준으로 그룹화하고, 각 연령대에 속한 아이돌 수를 조회
    NumberExpression<Integer> ageGroup = new CaseBuilder()
            .when(idol.age.between(10, 19)).then(10) // 10대
            .when(idol.age.between(20, 29)).then(20) // 20대
            .when(idol.age.between(30, 39)).then(30) // 30대
            .otherwise(0); // 그 외

    // when
    List<Tuple> result = factory
            .select(ageGroup, idol.count())
            .from(idol)
            .groupBy(ageGroup) // CaseBuilder로 만든 기준으로 그룹화
            .having(idol.count().goe(2)) // 그룹 크기가 2 이상인 경우만
            .fetch();

    // then
    // ... (결과 출력)
}
```

-----

## 2\. QueryDSL Join

여러 엔터티를 연결하여 데이터를 조회하는 `Join`에 대해 학습합니다.

### 📜 `Album.java` - 연관관계 엔터티

`Group`과 다대일(`@ManyToOne`) 관계를 가지는 `Album` 엔터티입니다.

```java
package com.spring.database.querydsl.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "tbl_album")
@Getter @Setter @ToString(exclude = {"group"})
@NoArgsConstructor @AllArgsConstructor
@Builder
public class Album {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "album_id")
    private Long id;

    private String albumName; // 앨범명
    private int releaseYear; // 발매연도

    // Album(N) : Group(1)
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "group_id", nullable = false)
    private Group group;

    // ... 생성자
}
```

### 📜 `QueryDslJoinTest.java`

조인 관련 테스트 코드입니다.

### 2-1. 기본 Join

#### Inner Join

두 엔터티에 모두 데이터가 존재하는 경우만 조회합니다. (그룹이 없는 아이돌은 제외)

```java
@Test
@DisplayName("Inner Join 예제")
void innerJoinTest () {
    // ...
    List<Idol> idolList = factory
            .select(idol)
            .from(idol)
            // innerJoin(조인 대상, 조인할 Q-Type)
            .innerJoin(idol.group, group)
            // fetchJoin(): N+1 문제를 해결하기 위해 연관된 엔터티를 함께 조회
            .fetchJoin()
            .fetch();
    // ... (결과 출력)
}
```

#### Left Outer Join

왼쪽 엔터티(`from` 절)를 기준으로, 오른쪽 엔터티에 매칭되는 데이터가 없어도 조회합니다. (그룹이 없는 아이돌도 포함)

```java
@Test
@DisplayName("Left Outer Join 예제")
void leftJoinTest() {
    // ...
    List<Idol> idolList = factory
            .select(idol)
            .from(idol)
            .leftJoin(idol.group, group) // leftJoin으로 변경
            .fetchJoin()
            .fetch();

    // ... (결과 출력, 그룹이 null인 경우 처리)
}
```

### 📌 `fetchJoin()`

`fetchJoin()`은 **N+1 문제**를 해결하기 위한 중요한 최적화 방법입니다.

  - **미사용 시**: `select` 쿼리로 아이돌 목록을 가져온 후, 각 아이돌의 그룹 정보를 얻기 위해 아이돌 수만큼 추가 쿼리(`N`개)가 발생합니다.
  - **사용 시**: `select` 쿼리를 실행할 때 연관된 엔터티(여기서는 `Group`)를 함께 조회하여 하나의 쿼리로 모든 정보를 가져옵니다.

### 2-2. 다중 Join

세 개 이상의 엔터티를 조인하는 예제입니다.

```java
@Test
@DisplayName("2022년에 발매된 앨범이 있는 아이돌의 정보 조회")
void albumTest() {
    /* SQL 쿼리
        SELECT I.idol_name, G.group_name, A.album_name, A.release_year
        FROM tbl_idol I
        JOIN tbl_group G ON I.group_id = G.group_id
        JOIN tbl_album A ON G.group_id = A.group_id
        WHERE A.release_year = 2022;
    */
    //given
    int year = 2022;
    //when
    List<Tuple> idolList = factory
            .select(idol, album)
            .from(idol)
            // 1. idol과 group 조인
            .innerJoin(idol.group, group)
            // 2. group과 album 조인
            // (Group 엔터티에는 `private List<Album> albums` 필드가 있어야 함)
            .innerJoin(group.albums, album)
            .where(album.releaseYear.eq(year))
            .fetch();
    //then
    for (Tuple tuple : idolList) {
        Idol foundIdol = tuple.get(idol);
        Album foundAlbum = tuple.get(album);
        System.out.printf("\n# 아이돌명: %s, 그룹명: %s, 앨범명: %s, 발매년도: %d년\n\n"
                , foundIdol.getIdolName(), foundIdol.getGroup().getGroupName()
                , foundAlbum.getAlbumName(), foundAlbum.getReleaseYear()
        );
    }
}
```
