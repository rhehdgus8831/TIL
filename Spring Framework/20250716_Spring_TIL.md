# Spring Boot REST API 

##  핵심 개념: 로깅(Logging)이란?

**로깅은 프로그램이 실행되는 과정에서 일어나는 일들을 기록하는 것입니다.**

### 로깅을 왜 사용하나요?
- **문제 해결**: 프로그램에 오류가 생겼을 때 어디서 문제가 발생했는지 찾을 수 있어요
- **모니터링**: 서버가 잘 돌아가고 있는지 확인할 수 있어요
- **기록 보관**: 누가 언제 무엇을 했는지 기록을 남길 수 있어요

### 로그 레벨 이해하기 (중요도 순서)

| 레벨 | 중요도 | 설명 | 실제 사용 예시 |
|------|--------|------|----------------|
| **TRACE** | 가장 낮음 | 메서드 호출, 변수 생성 등 가장 자세한 정보 | `log.trace("memberList 메서드 호출됨")` |
| **DEBUG** | 낮음 | 개발할 때 변수값 확인용 | `log.debug("members.size = {}", responses.size())` |
| **INFO** | 보통 | 서버 시작, 중요한 요청 등 일반적인 정보 | `log.info("/api/v5-3/members : GET - 요청 시작!")` |
| **WARN** | 높음 | 문제는 아니지만 이상한 상황 | `log.warn("회원 데이터가 없습니다.")` |
| **ERROR** | 매우 높음 | 프로그램에 문제가 생김 | `log.error("서버 에러입니다.")` |
| **FATAL** | 최고 | 시스템 전체 장애 | 시스템 전체 중단 상황 |

### 로그 설정하기
```properties
# application.properties
spring.application.name=spring-web-basic202507
server.port=9000

# 전체 로그 레벨을 INFO로 설정
logging.level.root = INFO
# 우리가 만든 패키지는 DEBUG 레벨까지 보여줌
logging.level.com.spring.basic = DEBUG

spring.mvc.view.prefix=/WEB-INF/views/
spring.mvc.view.suffix=.jsp
```

##  핵심 개념: Bean Validation (입력값 검증)

**사용자가 입력한 데이터가 올바른지 자동으로 검사하는 기능입니다.**

### 왜 검증이 필요한가요?
- **보안**: 악의적인 데이터 입력 방지
- **데이터 품질**: 잘못된 형식의 데이터 저장 방지
- **사용자 경험**: 입력 오류 시 친절한 안내 메시지 제공

### @Valid 어노테이션

| 구분 | 설명 | 사용 위치 |
|------|------|----------|
| **기본 개념** | JSR-303 표준 스펙으로 빈 검증기(Bean Validator)를 이용해 객체의 제약 조건을 검증 | 메서드 파라미터, 필드 |
| **주요 역할** | 컨트롤러에서 클라이언트 요청 데이터의 유효성을 자동으로 검증 | `@RequestBody @Valid MemberCreateDto dto` |
| **동작 원리** | ArgumentResolver에 의해 유효성 검사 진행 | Controller 메서드 파라미터 앞에 붙임 |
| **오류 처리** | 검증 실패 시 `MethodArgumentNotValidException` 발생 | `BindingResult`로 오류 정보 수집 |

### 주요 검증 어노테이션 설명

| 어노테이션 | 용도 | 설명 | 사용 예시 |
|-----------|------|------|----------|
| **@NotBlank** | 필수 입력 검증 | null, 빈 문자열(""), 공백(" ") 모두 허용하지 않음 | `@NotBlank(message = "닉네임은 필수입니다.")` |
| **@NotEmpty** | 비어있지 않음 검증 | 공백(" ")은 허용하되, null과 빈 문자열("")은 허용하지 않음 | `@NotEmpty(message = "내용을 입력하세요.")` |
| **@NotNull** | null 체크 | null값만 허용하지 않음 (빈 문자열, 공백은 허용) | `@NotNull(message = "필수 값입니다.")` |
| **@Email** | 이메일 형식 검증 | 이메일 형식이어야 함 (예: user@example.com) | `@Email(message = "이메일 형식을 지켜주세요.")` |
| **@Size** | 길이 검증 | 최소/최대 길이 제한 | `@Size(min=8, message = "8자리 이상 작성하세요.")` |
| **@Pattern** | 정규식 검증 | 정규표현식 패턴에 맞아야 함 | 비밀번호 복잡도 체크 |
| **@Min/@Max** | 숫자 범위 검증 | 최소/최대 값 제한 | `@Min(value = 18, message = "18세 이상이어야 합니다.")` |

### 회원가입 요청 데이터 클래스
```java
package com.spring.basic.chap5_3.dto.request;

import com.spring.basic.chap3_2.entity.Member;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;
import lombok.*;

// 회원 가입할 때 사용자가 입력하는 데이터를 담는 클래스
@Getter @Setter @ToString @EqualsAndHashCode
@NoArgsConstructor @AllArgsConstructor @Builder
public class MemberCreateDto {

    // 계정명 검증: 이메일 형식이어야 하고, 비어있으면 안 됨
    @Email(message = "계정명은 이메일 형식을 지켜주세요.")
    @NotBlank(message = "계정명은 필수입니다.")
    private String userAcc;

    // 비밀번호 검증: 8자리 이상, 특수문자/숫자/영문 포함
    @NotBlank(message = "비밀번호는 필수입니다.")
    @Size(min = 8, message = "8자리 이상으로 작성하세요.")
    @Pattern(regexp = "^(?=.*[A-Z])(?=.*[0-9])(?=.*[a-z])(?=.*[!@#$%^&*()-+=]).{8,}$", 
             message = "비밀번호는 특수문자와 숫자 영문을 포함해야 합니다.")
    private String pw;

    // 닉네임 검증: 비어있으면 안 됨
    @NotBlank(message = "닉네임은 필수입니다.")
    private String nick;

    // 입력받은 데이터를 Member 객체로 변환하는 메서드
    public static Member from(MemberCreateDto dto) {
        return Member.builder()
                .account(dto.getUserAcc())
                .password(dto.getPw())
                .nickname(dto.getNick())
                .build();
    }
}
```

##  핵심 개념: Jackson 어노테이션 (JSON 응답 커스터마이징)

**클라이언트에게 보내는 JSON 데이터의 형태를 자유롭게 조정할 수 있습니다.**

### Jackson 어노테이션의 역할

| 어노테이션 | 용도 | 설명 | 사용 예시 |
|-----------|------|------|----------|
| **@JsonProperty** | JSON 키 이름 변경 | JSON에서 필드 이름을 바꿔서 보여줌 | `@JsonProperty("account")` |
| **@JsonIgnore** | 필드 숨김 | 특정 필드를 JSON에서 숨김 (보안상 중요) | `@JsonIgnore private String cardNo;` |
| **@JsonFormat** | 날짜 형식 지정 | 날짜나 시간을 원하는 형태로 표시 | `@JsonFormat(pattern = "yyyy년-MM월-dd일")` |

### 보안을 위한 데이터 처리
- **마스킹**: 민감한 정보를 일부만 보여줌 (예: 홍*동)
- **필드 제외**: 카드번호 같은 민감 정보는 아예 안 보여줌

### 회원 목록 응답 데이터 클래스
```java
package com.spring.basic.chap5_4.dto.response;

import com.fasterxml.jackson.annotation.JsonFormat;
import com.fasterxml.jackson.annotation.JsonIgnore;
import com.fasterxml.jackson.annotation.JsonProperty;
import com.spring.basic.chap3_2.entity.Member;
import lombok.*;

import java.time.LocalDate;

// 클라이언트에게 회원 목록을 보내줄 때 사용하는 클래스
@Getter @Setter @ToString @EqualsAndHashCode
@NoArgsConstructor @AllArgsConstructor @Builder
public class MemberListResponse {

    private String id;

    // JSON에서는 "account"라는 이름으로 보여짐
    @JsonProperty("account")
    private String email;

    // 닉네임은 마스킹 처리됨 (첫글자 + * + 마지막글자)
    private String nick;

    // 이 필드는 JSON 응답에서 완전히 숨겨짐
    @JsonIgnore
    private String cardNo;

    // 날짜를 "2025년-01월-15일" 형태로 표시
    @JsonFormat(pattern = "yyyy년-MM월-dd일")
    private LocalDate creationTime;

    // Member 객체를 받아서 응답용 객체로 변환
    public static MemberListResponse from(Member member) {
        return MemberListResponse.builder()
                .id(member.getUid())
                .email(member.getAccount())
                .nick(maskingNickName(member.getNickname())) // 닉네임 마스킹
                .creationTime(LocalDate.now())
                .build();
    }

    // 닉네임을 마스킹하는 메서드 (예: "홍길동" -> "홍*동")
    private static String maskingNickName(String originNick) {
        return originNick.charAt(0) + "*" + originNick.charAt(originNick.length() - 1);
    }
}
```

##  핵심 개념: REST Controller 구현

**클라이언트의 HTTP 요청을 받아서 처리하고, 적절한 응답을 돌려주는 역할을 합니다.**

### REST Controller의 주요 역할
1. **요청 받기**: 클라이언트의 HTTP 요청을 받음
2. **데이터 검증**: 받은 데이터가 올바른지 확인
3. **비즈니스 로직 처리**: 실제 작업 수행
4. **응답 반환**: 결과를 클라이언트에게 보냄

### HTTP 상태 코드 이해

| 상태 코드 | 의미 | 설명 | 사용 상황 |
|-----------|------|------|----------|
| **200 OK** | 성공 | 요청이 성공적으로 처리됨 | 정상적인 데이터 조회/처리 |
| **400 Bad Request** | 클라이언트 오류 | 클라이언트 요청에 문제가 있음 | 검증 실패, 잘못된 요청 형식 |
| **404 Not Found** | 리소스 없음 | 요청한 리소스를 찾을 수 없음 | 존재하지 않는 회원 조회 |
| **500 Internal Server Error** | 서버 오류 | 서버에 예상치 못한 오류가 발생함 | 시스템 예외, 서버 장애 |

### 회원 관리 REST Controller
```java
package com.spring.basic.chap5_3.controller;

import com.spring.basic.chap3_2.entity.Member;
import com.spring.basic.chap5_3.dto.request.MemberCreateDto;
import com.spring.basic.chap5_4.dto.response.MemberListResponse;
import jakarta.validation.Valid;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

import static java.util.stream.Collectors.toList;

@RestController // 이 클래스가 REST API를 처리하는 컨트롤러임을 표시
@RequestMapping("/api/v5-3/members") // 기본 URL 경로 설정
@Slf4j // 로그를 사용할 수 있게 해주는 어노테이션
public class MemberController5_3 {

    // 임시로 회원 정보를 저장할 Map (실제로는 데이터베이스 사용)
    private Map memberStore = new HashMap<>();

    // 컨트롤러 생성 시 테스트용 데이터 준비
    public MemberController5_3() {
        Member m1 = Member.builder()
                .account("abc1234")
                .password("12345678")
                .nickname("호돌이")
                .build();

        Member m2 = Member.builder()
                .account("def9876")
                .password("12345678")
                .nickname("핑돌이")
                .build();

        memberStore.put(m1.getAccount(), m1);
        memberStore.put(m2.getAccount(), m2);
    }

    // GET 요청 처리: 전체 회원 목록 조회
    @GetMapping
    public ResponseEntity> memberList() {
        log.trace("memberList 메서드 호출됨"); // 메서드 시작 로그
        log.info("/api/v5-3/members : GET - 요청 시작!"); // 요청 처리 시작 로그

        // 저장된 회원들을 응답용 객체로 변환
        List responses = memberStore.values()
                .stream()
                .map(MemberListResponse::from) // 각 Member를 MemberListResponse로 변환
                .collect(toList());

        log.debug("members.size = {}", responses.size()); // 회원 수 로그

        // 회원이 없으면 404 Not Found 응답
        if (responses.size()  create(
            @RequestBody @Valid MemberCreateDto dto, // @Valid로 유효성 검증 활성화
            BindingResult bindingResult // 검증 결과를 담는 객체
    ) {
        // 검증 오류가 있는 경우 처리
        if (bindingResult.hasErrors()) {
            Map errorMap = new HashMap<>();
            
            // 각 필드별 오류 메시지 수집
            bindingResult.getFieldErrors().forEach(err -> {
                errorMap.put(err.getField(), err.getDefaultMessage());
            });

            log.warn("회원가입 입력값 오류가 발생함!");
            // 400 Bad Request 응답과 함께 오류 메시지 반환
            return ResponseEntity.badRequest().body(errorMap);
        }

        log.info("param - {}", dto); // 입력 파라미터 로그

        // DTO를 Entity로 변환
        Member member = MemberCreateDto.from(dto);
        
        // 메모리에 저장 (실제로는 데이터베이스에 저장)
        memberStore.put(dto.getUserAcc(), member);
        
        // 200 OK 응답과 함께 성공 메시지 반환
        return ResponseEntity.ok("created member: " + member);
    }
}
```

##  전체 흐름 이해하기

### 1. 회원 조회 흐름
1. 클라이언트가 `GET /api/v5-3/members` 요청
2. Spring Boot가 `memberList()` 메서드 호출
3. 저장된 회원 데이터를 `MemberListResponse`로 변환
4. JSON 형태로 응답 반환

### 2. 회원 생성 흐름 (@Valid 활용)
1. 클라이언트가 `POST /api/v5-3/members` 요청 (회원 정보 포함)
2. Spring Boot가 JSON을 `MemberCreateDto` 객체로 변환
3. **@Valid 어노테이션**이 Bean Validation을 활성화하여 입력값 검증 수행
4. 검증 실패 시 `BindingResult`에 오류 정보 수집
5. 검증 통과 시 `Member` 객체 생성 후 저장
6. 성공 메시지 반환

### 3. 로깅 활용
- **TRACE**: 메서드 진입/종료 추적
- **DEBUG**: 변수값 확인
- **INFO**: 중요한 요청 처리 기록
- **WARN**: 문제 상황 경고
- **ERROR**: 오류 발생 시 기록

이렇게 **@Valid 어노테이션**을 활용하면 안전하고 유지보수하기 쉬운 REST API를 만들 수 있습니다!

[1] https://velog.io/@minji/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-Spring-Boot%EC%9D%98-Validation-Valid-%EC%96%B4%EB%85%B8%ED%85%8C%EC%9D%B4%EC%85%98%EC%9C%BC%EB%A1%9C-%EC%9C%A0%ED%9A%A8%EC%84%B1-%EA%B2%80%EC%A6%9D%ED%95%98%EA%B8%B0
[2] https://jh4dev.tistory.com/44
[3] https://priming.tistory.com/128
[4] https://dev-coco.tistory.com/123
[5] https://dev-jwblog.tistory.com/144
[6] https://jh4dev.tistory.com/45
[7] https://kdhyo98.tistory.com/81
[8] https://softwaresaramdle.tistory.com/20
[9] https://zoetechlog.tistory.com/93
[10] https://mangkyu.tistory.com/174