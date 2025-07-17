# Spring Web MVC와 REST API 종합 예제 - 성적 관리 시스템

## 📚 개요

이 예제는 Spring Boot를 활용한 성적 관리 시스템으로, **REST API**와 **웹 MVC** 패턴을 통합적으로 보여주는 종합 예제입니다. 학생들의 성적 정보를 **CRUD**(Create, Read, Update, Delete) 방식으로 관리하며, **DTO 패턴**과 **유효성 검증**을 포함한 실무적인 구조를 가지고 있습니다.

## 🏗️ 프로젝트 구조 및 아키텍처

### 전체 패키지 구조
```
com.spring.basic.score/
├── api/
│   └── ScoreApiController.java      # REST API 컨트롤러
├── dto/
│   ├── request/
│   │   └── ScoreCreateRequest.java  # 요청 DTO
│   └── response/
│       ├── ScoreListResponse.java   # 목록 응답 DTO
│       └── ScoreDetailResponse.java # 상세 응답 DTO
├── entity/
│   └── Score.java                   # 엔티티 클래스
└── view/
    ├── score-page.jsp              # 목록 페이지
    └── score-detail.jsp            # 상세 페이지
```

### 아키텍처 패턴
- **MVC (Model-View-Controller)** 패턴
- **DTO (Data Transfer Object)** 패턴
- **Entity** 패턴
- **RESTful API** 설계

## 🔄 전체 코드 흐름도

### 1. 성적 등록 흐름
```
Client (POST /api/v1/scores)
    ↓ JSON 데이터
ScoreApiController.create()
    ↓ @Valid 유효성 검증
ScoreCreateRequest DTO
    ↓ toEntity() 변환
Score Entity
    ↓ 저장
HashMap scoreStore
    ↓ 응답
ResponseEntity
```

### 2. 성적 목록 조회 흐름
```
Client (GET /api/v1/scores?sort=average)
    ↓
ScoreApiController.scoreList()
    ↓ 저장소에서 조회
HashMap scoreStore
    ↓ 석차 계산
calculateListRank()
    ↓ DTO 변환
ScoreListResponse List
    ↓ 정렬 처리
getScoreComparator()
    ↓ JSON 응답
ResponseEntity>
```

### 3. 성적 상세 조회 흐름
```
Client (GET /api/v1/scores/{id})
    ↓
ScoreApiController.scoreDetail()
    ↓ ID로 조회
HashMap scoreStore
    ↓ 석차 계산
calculateListRank()
    ↓ DTO 변환
ScoreDetailResponse.of()
    ↓ JSON 응답
ResponseEntity
```

## 📁 상세 코드 분석

### 1. ScoreApiController - REST API 컨트롤러[1]

```java
@RestController // REST API 컨트롤러 선언
@RequestMapping("/api/v1/scores") // 기본 URL 패턴
@Slf4j // 로그 라이브러리
public class ScoreApiController {
    
    // 저장소 역할을 할 해시맵 (실제로는 데이터베이스 사용)
    private Map scoreStore = new HashMap<>();
    private Long nextId = 1L;
    
    // 생성자에서 테스트 데이터 초기화
    public ScoreApiController() {
        Score s1 = new Score(nextId++, "김말복", 100, 88, 75);
        Score s2 = new Score(nextId++, "박수포자", 55, 95, 15);
        Score s3 = new Score(nextId++, "김마이클", 99, 100, 90);
        Score s4 = new Score(nextId++, "세종대왕", 100, 0, 90);
        
        scoreStore.put(s1.getId(), s1);
        scoreStore.put(s2.getId(), s2);
        scoreStore.put(s3.getId(), s3);
        scoreStore.put(s4.getId(), s4);
    }
```

#### 주요 어노테이션 정리

| 어노테이션 | 설명 | 사용 위치 |
|------------|------|-----------|
| `@RestController` | REST API 컨트롤러 선언, `@Controller` + `@ResponseBody` | 클래스 |
| `@RequestMapping` | 기본 URL 패턴 설정 | 클래스 |
| `@PostMapping` | HTTP POST 요청 처리 | 메서드 |
| `@GetMapping` | HTTP GET 요청 처리 | 메서드 |
| `@DeleteMapping` | HTTP DELETE 요청 처리 | 메서드 |
| `@PathVariable` | URL 경로의 변수 값 추출 | 매개변수 |
| `@RequestParam` | 쿼리 파라미터 값 추출 | 매개변수 |
| `@RequestBody` | HTTP 요청 본문을 객체로 변환 | 매개변수 |
| `@Valid` | 유효성 검증 활성화 | 매개변수 |
| `@Slf4j` | 로그 객체 자동 생성 | 클래스 |

### 2. 성적 등록 API (POST)[1]

```java
@PostMapping
public ResponseEntity create(
    @RequestBody @Valid ScoreCreateRequest dto, // JSON을 DTO로 변환 + 유효성 검증
    BindingResult bindingResult // 유효성 검증 에러 정보
) {
    log.info("/api/v1/scores POST!");
    log.debug("param: score - {}", dto);
    
    // 입력값 검증에서 에러가 발생했다면
    if (bindingResult.hasErrors()) {
        Map errorMap = new HashMap<>();
        bindingResult.getFieldErrors().forEach(err -> {
            errorMap.put(err.getField(), err.getDefaultMessage());
        });
        return ResponseEntity
                .badRequest()
                .body(errorMap);
    }
    
    // 실제로 데이터베이스에 저장
    Score score = ScoreCreateRequest.toEntity(dto);
    score.setId(nextId++);
    scoreStore.put(score.getId(), score);
    
    return ResponseEntity.ok().body("성적 정보 생성 완료 - " + dto);
}
```

### 3. 성적 목록 조회 API (GET)[1]

```java
@GetMapping
public ResponseEntity scoreList(
    @RequestParam(defaultValue = "id") String sort // 정렬 기준 (기본값: id)
) {
    log.info("/api/v1/scores GET !");
    log.debug("param: sort - {}", sort);
    
    // 클라이언트에게 성적정보 목록을 JSON 반환
    List scores = new ArrayList<>(scoreStore.values());
    
    // 석차를 구하는 로직
    calculateListRank(scores);
    
    // 깔끔하게 응답데이터들을 모아둔 DTO로 변환
    List responses = scores.stream()
            .map(ScoreListResponse::from)
            .collect(Collectors.toList());
    
    // 정렬 처리
    responses.sort(getScoreComparator(sort));
    
    return ResponseEntity.ok().body(responses);
}
```

### 4. 석차 계산 로직[1]

```java
// 원본 리스트에서 석차를 구해서 세팅
private void calculateListRank(List scores) {
    // 총점 내림차순으로 정렬
    scores.sort(Comparator.comparing(Score::getTotal).reversed());
    
    // 순서대로 석차 부여
    for (int i = 0; i  getScoreComparator(String sort) {
    Comparator comparator = null;
    switch (sort) {
        case "id" -> comparator = Comparator.comparing(ScoreListResponse::getId);
        case "name" -> comparator = Comparator.comparing(ScoreListResponse::getMaskingName);
        case "average" -> comparator = Comparator.comparing(ScoreListResponse::getAvg).reversed();
    }
    return comparator;
}
```

## 📝 DTO (Data Transfer Object) 패턴

### 1. ScoreCreateRequest - 요청 DTO[2]

```java
@Getter @Setter @ToString @EqualsAndHashCode
@NoArgsConstructor @AllArgsConstructor @Builder
public class ScoreCreateRequest {
    
    // 이름 유효성 검증
    @NotEmpty(message = "이름은 필수값입니다.")
    @Pattern(regexp = "^[가-힣]+$", message = "이름은 한글로만 작성하세요!")
    private String studentName;
    
    // 국어 점수 유효성 검증
    @Min(value = 0, message = "국어 점수는 0점 이상이어야합니다.")
    @Max(value = 100, message = "국어 점수는 100점 이하여야합니다.")
    @NotNull(message = "국어점수는 필수값입니다.")
    private Integer korean;
    
    // 영어 점수 유효성 검증
    @Min(value = 0, message = "영어 점수는 0점 이상이어야합니다.")
    @Max(value = 100, message = "영어 점수는 100점 이하여야합니다.")
    @NotNull(message = "영어점수는 필수값입니다.")
    private Integer english;
    
    // 수학 점수 유효성 검증
    @Min(value = 0, message = "수학 점수는 0점 이상이어야합니다.")
    @Max(value = 100, message = "수학 점수는 100점 이하여야합니다.")
    @NotNull(message = "수학점수는 필수값입니다.")
    private Integer math;
    
    // DTO를 엔터티로 변환하는 정적 메서드
    public static Score toEntity(ScoreCreateRequest dto) {
        Score score = Score.builder()
                .name(dto.getStudentName())
                .kor(dto.getKorean())
                .math(dto.getMath())
                .eng(dto.getEnglish())
                .build();
        score.calcTotalAndAverage(); // 총점과 평균 계산
        return score;
    }
}
```

#### 유효성 검증 어노테이션 정리

| 어노테이션 | 설명 | 예시 |
|------------|------|------|
| `@NotEmpty` | null과 빈 문자열 모두 허용하지 않음 | 이름 필드 |
| `@NotNull` | null 값 허용하지 않음 | 점수 필드 |
| `@Pattern` | 정규표현식 패턴 검증 | 한글만 허용 |
| `@Min` | 최솟값 검증 | 점수 0점 이상 |
| `@Max` | 최댓값 검증 | 점수 100점 이하 |

### 2. ScoreListResponse - 목록 응답 DTO[3]

```java
@Getter @Setter @ToString @EqualsAndHashCode
@NoArgsConstructor @AllArgsConstructor @Builder
public class ScoreListResponse {
    
    private Long id;
    private String maskingName; // 마스킹된 이름
    
    @JsonProperty("sum") // JSON 필드명을 "sum"으로 변경
    private int total;
    
    private double avg;
    private int rank;
    
    // 엔터티를 DTO로 변환하는 정적 메서드
    public static ScoreListResponse from(Score score) {
        return ScoreListResponse.builder()
                .id(score.getId())
                .total(score.getTotal())
                .avg(score.getAverage())
                .maskingName(convertMaskingName(score.getName()))
                .rank(score.getRank())
                .build();
    }
    
    // 이름 마스킹 처리 로직
    private static String convertMaskingName(String originName) {
        // 이름이 2글자인 사람 처리
        if (originName.length() 

```

### 2. score-detail.jsp - 상세 페이지[6]

```jsp


```

## 🔧 핵심 개념 정리

### 1. REST API 설계 원칙

| HTTP 메서드 | URL 패턴 | 기능 | 응답 형태 |
|-------------|----------|------|-----------|
| GET | `/api/v1/scores` | 전체 목록 조회 | JSON 배열 |
| GET | `/api/v1/scores/{id}` | 개별 상세 조회 | JSON 객체 |
| POST | `/api/v1/scores` | 새 데이터 등록 | 성공 메시지 |
| DELETE | `/api/v1/scores/{id}` | 데이터 삭제 | 성공 메시지 |

### 2. DTO 패턴의 장점

- **데이터 은닉**: 엔터티의 모든 필드를 노출하지 않음
- **유효성 검증**: 클라이언트 입력값 검증
- **데이터 변환**: 엔터티와 클라이언트 간 데이터 형태 변환
- **버전 관리**: API 버전별로 다른 DTO 사용 가능

### 3. ResponseEntity 활용

```java
// 성공 응답
return ResponseEntity.ok().body(responseData);

// 실패 응답 (400 Bad Request)
return ResponseEntity.badRequest().body("에러 메시지");

// 커스텀 상태 코드
return ResponseEntity.status(HttpStatus.CREATED).body(data);
```

### 4. 유효성 검증 처리 흐름

```
@Valid 어노테이션
    ↓
Bean Validation 수행
    ↓
BindingResult에 에러 저장
    ↓
hasErrors() 체크
    ↓
에러 메시지 응답 또는 정상 처리
```

## 💡 실무 팁

### 1. 에러 처리 패턴
- **BindingResult**를 활용한 유효성 검증 에러 처리
- **ResponseEntity**를 통한 HTTP 상태 코드 명시적 관리
- 클라이언트 친화적인 에러 메시지 제공

### 2. 데이터 보안
- **이름 마스킹 처리**로 개인정보 보호
- 상세 조회와 목록 조회에서 다른 정보 노출 수준

### 3. 성능 최적화
- **Stream API**를 활용한 함수형 프로그래밍
- **Method Reference**를 통한 간결한 코드
- **정렬 로직** 분리로 재사용성 향상
