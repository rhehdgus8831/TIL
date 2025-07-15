
# **Spring RESTful API 심화: 전체 코드 및 개념 총정리**

오늘은 기본 CRUD API를 넘어, 실무에서 사용하는 고급 REST API 개발 기술들을 학습했다./
JSON 요청 데이터를 객체로 변환하는 **`@RequestBody`**, 객체 생성을 위한 **빌더 패턴**, HTTP 응답을 직접 제어하는 **`ResponseEntity`**,\
그리고 웹 보안 정책인 **CORS**에 대해 정리한다.

-----

## **1. 핵심 어노테이션 정리**

| 어노테이션 | 역할 및 설명 |
| :--- | :--- |
| **`@RequestBody`** | 요청 본문(body)에 담긴 JSON 데이터를 자바 객체로 자동 변환하여 받아준다. |
| **`@Builder` (Lombok)**| 객체 생성을 위한 빌더 패턴 코드를 자동으로 생성하여 가독성과 편의성을 높인다. |

-----

## **2. 빌더 패턴(Builder Pattern)과 `@Builder`**

### **2-1. 개념**

**빌더 패턴**은 복잡한 객체를 생성하는 방법을 정의하는 디자인 패턴이다.\
생성자에 파라미터가 너무 많거나, 일부 필드만 선택적으로 초기화하고 싶을 때 사용하면 코드의 가독성이 매우 좋아진다.\
Lombok의 `@Builder` 어노테이션을 사용하면 이 패턴을 매우 쉽게 적용할 수 있다.

### **2-2. `Member.java` 전체 코드 및 분석**

```java
package com.spring.basic.chap3_2.entity;

@Getter @Setter @ToString
@EqualsAndHashCode
@NoArgsConstructor
@AllArgsConstructor
@Builder // Lombok에게 빌더 패턴 코드를 자동으로 생성하라고 지시
public class Member {

    @Builder.Default // 빌더를 통해 객체 생성 시, 값을 주지 않으면 이 기본값으로 설정됨
    private String  uid = UUID.randomUUID().toString(); // 회원식별번호
    private String account;
    private String password;
    private String nickname;

    /* // @Builder 어노테이션이 아래의 코드를 자동으로 만들어 줌
    public Member(Builder builder) {
        this.uid = builder.uid;
        this.account = builder.account;
        this.password = builder.password;
        this.nickname = builder.nickname;
    }

    // 빌더 패턴 구현 - 생성자를 대체하는 것
    public static class Builder{
        // 원본 클래스랑 완벽하게 동일한 필드를 구성
        private String  uid;
        private String account;
        private String password;
        private String nickname;

        public Builder() {}

        // 필드를 초기화하는 setter를 자기 필드명가 동일하게 생성
        public Builder account(String account) {
            // 자기 자신 객체를 리턴
            this.account = account;
            return this;
        }
        public Builder password(String password) {
            // 자기 자신 객체를 리턴
            this.password = password;
            return this;
        }
        public Builder nickname(String nickname) {
            // 자기 자신 객체를 리턴
            this.nickname = nickname;
            return this;
        }

        // 최종 연산에서는 원복 객체를 리턴
        public Member build() {
            return new Member(this);
        }
    }
    */
}
```

-----

## **3. JSON 요청 처리: `@RequestBody`**

### **3-1. `MemberController3_2.java` 전체 코드 및 분석**

* **개념**: `@RequestBody`는 클라이언트가 요청 본문(body)에 담아 보낸 JSON 문자열을 스프링이 내부적으로 `Member` 자바 객체로 자동 변환(역직렬화)하여 파라미터로 전달해주는 강력한 기능이다.

<!-- end list -->

```java
package com.spring.basic.chap3_2.controller;

@RestController
@RequestMapping("/api/v3-2/members")
public class MemberController3_2 {

    // 회원 정보를 저장할 임시 데이터 저장소
    private Map<String, Member> memberStore = new HashMap<>();

    // 컨트롤러 생성 시 초기 멤버 데이터 추가
    public MemberController3_2() {
        Member member1 = Member.builder()
                .account("abc123")
                .password("9999")
                .nickname("뽀롱이")
                .build();

        Member member2 = Member.builder()
                .account("def9876")
                .password("4444")
                .nickname("핑순이")
                .build();

        memberStore.put(member1.getUid(), member1);
        memberStore.put(member2.getUid(), member2);
    }

    // 전체 조회
    @GetMapping
    public List<Member> list() {
        return new ArrayList<>(memberStore.values());
    }

    // 회원 등록
    // ?account=xxx&password=xxx&nickname=xxx   -> 불편
    // 전송할 데이터를 JSON객체로 묶어서 보내줘 내가 풀어서 쓸게
    /*
        {
            "account": "xxx",
            "password": "xxx",
            "nickname": "xxx",
        }
     */
    @PostMapping
    public String join(@RequestBody Member member) { // 요청 본문의 JSON을 Member 객체로 매핑
        member.setUid(UUID.randomUUID().toString());
        memberStore.put(member.getUid(), member);
        return "새로운 멤버가 생성됨! - nickname : " + member.getNickname();
    }
}
```

-----

## **4. 응답 커스터마이징: `ResponseEntity`**

### **4-1. `MemberController3_3.java` 전체 코드 및 분석**

* **개념**: `ResponseEntity` 객체를 사용하면 \*\*HTTP 응답 상태 코드(Status Code), 헤더(Headers), 응답 본문(Body)\*\*을 모두 직접 제어할 수 있다.\
이를 통해 클라이언트에게 더 정확하고 상세한 응답 정보를 제공할 수 있다.

<!-- end list -->

```java
package com.spring.basic.chap3_3.controller;

@RestController
@RequestMapping("/api/v3-3/members")
// @CrossOrigin("http://127.0.0.1:5500") // 개별 컨트롤러 CORS 설정 (전역 설정 권장)
public class MemberController3_3 {

    private Map<String, Member> memberStore = new HashMap<>();

    public MemberController3_3() {
        Member member1 = Member.builder()
                .account("abc123")
                .password("9999")
                .nickname("뽀롱이")
                .build();

        Member member2 = Member.builder()
                .account("def9876")
                .password("4444")
                .nickname("핑순이")
                .build();

        memberStore.put(member1.getUid(), member1);
        memberStore.put(member2.getUid(), member2);
    }

    // 회원 생성
    @PostMapping
    public ResponseEntity<String> join(@RequestBody Member member) {
        // 계정이 비어있는 지 확인
        if (member.getAccount().isBlank()) {
            // 400 Bad Request 상태 코드와 함께 에러 메시지를 응답
            return ResponseEntity
                    .badRequest()
                    .body("계정은 필수값입니다.");
        }
        member.setUid(UUID.randomUUID().toString());
        memberStore.put(member.getUid(), member);
        // 200 OK 상태 코드와 함께 성공 메시지를 응답
        return ResponseEntity
                .ok()
                .body("새로운 멤버가 생성됨! - nickname : " + member.getNickname());
    }

    // 전체 조회
    @GetMapping
    public ResponseEntity<?> list() {
        /*try {
            String str = null;
            str.charAt(0);
        } catch (Exception e) {
            return ResponseEntity
                    .internalServerError()
                    .body("서버측 에러입니다 ㅈㅅ");
        }*/

        // 응답 헤더에 커스텀 정보 추가
        HttpHeaders httpHeaders = new HttpHeaders();
        httpHeaders.add("my-pet", "cat");
        httpHeaders.add("hobby", "swimming");

        // 200 OK 상태 코드, 커스텀 헤더, 회원 목록 데이터를 담아 응답
        return ResponseEntity
                .ok()
                .headers(httpHeaders)
                .body(new ArrayList<>(memberStore.values()));
    }

    // 단일 조회 (계정명으로 조회)
    @GetMapping("/{account}")
    public ResponseEntity<?> findOne(@PathVariable String account) {
        Member foundMember = null;
        for (String key : memberStore.keySet()) {
            Member member = memberStore.get(key);
            if (member.getAccount().equals(account)) {
                foundMember = member;
            }
        }
        
        // Member foundMember = memberStore.values()
        //         .stream()
        //         .filter(member -> member.getAccount().equals(account))
        //         .findFirst()
        //         .orElse(null);

        if (foundMember == null) {
            // 404 Not Found 상태 코드와 함께 에러 메시지를 응답
            return ResponseEntity
                    .status(404)
                    .body(account + "는(은) 존재하지 않는 계정입니다.");
        }
        // 200 OK 상태 코드와 함께 찾은 회원 정보를 응답
        return ResponseEntity.ok().body(foundMember);
    }
}
```

-----

## **5. CORS (Cross-Origin Resource Sharing)**

### **5-1. `CorsConfig.java` 전체 코드 및 분석**

* **개념**: **CORS**는 웹 보안 정책인 \*\*SOP(Same-Origin Policy, 동일 출처 정책)\*\*으로 인해 발생하는 문제를 해결하기 위한 메커니즘이다.\
기본적으로 웹 브라우저는 다른 출처(도메인, 포트 등이 다른 경우)의 리소스를 요청하는 것을 차단한다.\
이 코드는 서버가 특정 출처의 클라이언트에게 리소스 접근을 허용하도록 명시하는 전역 설정이다.

<!-- end list -->

```java
package com.spring.basic.chap3_4.config;

// 전역 크로스 오리진 설정 : 특정 클라이언트가 API를 사용하기 위한 정책 설정
@Configuration // 스프링의 설정 파일임을 명시
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry
                // 1. 어떤 URL 요청에 대해 CORS를 허용할 것인가
                .addMapping("/api/v3-3/*")
                // 2. 어떤 클라이언트(출처)를 허용할 것인가
                .allowedOrigins("http://127.0.0.1:5500")
                // 3. 어떤 요청 방식(GET, POST...)을 허용할 것인가
                .allowedMethods("GET","POST")
                // 4. 어떤 요청 헤더를 허용할 것인가
                .allowedHeaders("*"); // 모든 헤더를 허용
    }
}
```

* **분석**: 위 설정은 `http://127.0.0.1:5500`에서 오는 요청 중,\
`/api/v3-3/`로 시작하는 URL에 대한 `GET`, `POST` 요청을 허용하겠다는 의미다.