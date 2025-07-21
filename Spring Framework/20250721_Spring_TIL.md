
# 스프링 예외처리(Exception Handling)

##  개요

스프링에서는 **체계적이고 일관성 있는 예외처리**를 위해 다음과 같은 구조를 제공합니다:
- **사용자 정의 예외 클래스** 생성
- **에러 코드 통합 관리**
- **전역 예외 처리기** 구현
- **일관성 있는 에러 응답 형식** 정의

##  1. 사용자 정의 예외 클래스 (Custom Exception)

### MemberException.java
```java
package com.spring.basic.chap6.exception.custom;

import com.spring.basic.chap6.exception.dto.ErrorCode;
import lombok.Getter;

@Getter
// 사용자 정의 예외 클래스 - 개발자가 원하는 스펙에 맞는 에러 클래스
public class MemberException extends RuntimeException{

    private final int status; // 에러 상태 코드와 이름을 포함
    private final String errorName; // 에러의 이름

    public MemberException(ErrorCode errorCode) {
        super(errorCode.getErrorMessage());
        this.errorName = errorCode.toString();
        this.status = errorCode.getStatusCode();
    }

}
```

### 핵심 포인트
- **RuntimeException을 상속**하여 언체크드 예외로 구현
- **ErrorCode enum을 받아** 에러 정보를 자동으로 설정
- **status, errorName 필드**를 통해 구체적인 에러 정보 보관

##  2. 에러 코드 관리 (ErrorCode Enum)

### ErrorCode.java
```java
package com.spring.basic.chap6.exception.dto;

import lombok.AllArgsConstructor;

@lombok.Getter
@AllArgsConstructor
public enum ErrorCode {
    MEMBER_NOT_FOUND(404,"회원을 찾을 수 없습니다."),
    ACCOUNT_TOO_LONG(400,"계정명은 10글자 이내여야합니다.");

    private final int statusCode;
    private final String errorMessage;

}
```

### 에러 코드 관리의 장점
- **중앙 집중식 에러 관리**: 모든 에러 코드를 한 곳에서 관리
- **타입 안전성**: enum을 사용하여 컴파일 타임에 오타 방지
- **유지보수성**: 에러 메시지 변경 시 한 곳만 수정하면 됨

##  3. 에러 응답 DTO (ErrorResponse)

### ErrorResponse.java
```java
package com.spring.basic.chap6.exception.dto;

import lombok.*;

import java.time.LocalDateTime;

@EqualsAndHashCode
@Getter
@Setter
@ToString
@AllArgsConstructor
@NoArgsConstructor
@Builder
// 에러 응답 시 사용할 형식있는 객체
public class ErrorResponse {

    private LocalDateTime timestamp ; // 에러가 발생한 시간
    private int status; // 에러 상태코드
    private String error; // 에러의 이름
    private String message; // 에러의 원인 메세지
    private String path; // 에러가 발생한 URL 정보
}
```

### ErrorResponse 구성 요소

| 필드 | 타입 | 설명 |
|------|------|------|
| `timestamp` | LocalDateTime | 에러 발생 시각 |
| `status` | int | HTTP 상태 코드 (404, 400 등) |
| `error` | String | 에러의 이름/타입 |
| `message` | String | 사용자에게 보여줄 에러 메시지 |
| `path` | String | 에러가 발생한 API 경로 |

##  4. 전역 예외 처리기 (GlobalExceptionHandler)

### GlobalExceptionHandler.java
```java
package com.spring.basic.chap6.exception;

import com.spring.basic.chap6.exception.custom.MemberException;
import com.spring.basic.chap6.exception.dto.ErrorResponse;
import jakarta.servlet.http.HttpServletRequest;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;

import java.time.LocalDateTime;

// API에서 발생하는 수많은 예외상황들을 도맡아서 처리하는 클래스
@Slf4j
@ControllerAdvice  // 컨트롤러 대신 예외처리를 하겠다
public class GlobalExceptionHandler {

    // 예외처리 메서드
    @ExceptionHandler(MemberException.class)
    public ResponseEntity handleClientException(
            MemberException e
            , HttpServletRequest request

    ) {
        // 로깅 처리
        log.warn("exception occurred! caused by: {}", e.getMessage());

        // 구체적인 에러 정보 객체를 생성
        ErrorResponse errorResponse = ErrorResponse.builder()
                .status(e.getStatus())
                .error(e.getErrorName())
                .message(e.getMessage())
                .path(request.getRequestURI())
                .timestamp(LocalDateTime.now())
                .build();

        // 에러 응답 처리
        return ResponseEntity.status(e.getStatus()).body(errorResponse);
    }

    // 입력값 검증 에러 통합 처리
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity handleBindingError(MethodArgumentNotValidException e,HttpServletRequest request){
        ErrorResponse errorResponse = ErrorResponse.builder()
                .path(request.getRequestURI())
                .timestamp(LocalDateTime.now())
                .status(e.getStatusCode().value())
                .error(e.getStatusCode().toString())
                .message(e.getBindingResult().getFieldError().getDefaultMessage())
                .build();

        return ResponseEntity.status(e.getStatusCode().value()).body(errorResponse);

    }

}
```

### 주요 어노테이션

| 어노테이션 | 설명 | 사용 위치 |
|-----------|------|----------|
| `@ControllerAdvice` | 전역 예외 처리 클래스임을 명시 | 클래스 레벨 |
| `@ExceptionHandler` | 특정 예외 타입을 처리할 메서드 지정 | 메서드 레벨 |

### 처리하는 예외 타입
1. **MemberException**: 사용자 정의 예외
2. **MethodArgumentNotValidException**: `@Valid` 검증 실패 시 발생하는 예외

##  5. 데이터 저장소 (Repository)

### MemberRepository.java
```java
package com.spring.basic.chap6.repository;

import com.spring.basic.chap3_2.entity.Member;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Repository;

import java.util.HashMap;
import java.util.Map;

@Slf4j
@Repository
// 데이터 베이스를 관리 CRUD
public class MemberRepository {

    // 데이터 베이스 정의
    private Map memberStore = new HashMap<>();

    public MemberRepository() {
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

    // 개별 조회하는 기능
    public Member findByAccount(String account){
        log.info("서비스로부터 회원 개별 조회를 위임받음");
        return memberStore.get(account);
    }

    // 데이터를 저장하는 기능
    public Member save(Member member){
        memberStore.put(member.getAccount(), member);
        return findByAccount(member.getAccount());
    }

}
```

### Repository 핵심 포인트
- **Map을 사용한 메모리 데이터베이스** 구현
- **생성자에서 초기 데이터** 설정 (호돌이, 핑돌이)
- **findByAccount()**: 계정명으로 회원 조회
- **save()**: 회원 정보 저장 후 저장된 회원 반환

## 🏗️ 6. 서비스 계층에서의 예외 발생

### MemberService.java
```java
package com.spring.basic.chap6.service;

import com.spring.basic.chap3_2.entity.Member;
import com.spring.basic.chap5_3.dto.request.MemberCreateDto;
import com.spring.basic.chap5_4.dto.response.MemberListResponse;
import com.spring.basic.chap6.exception.custom.MemberException;
import com.spring.basic.chap6.exception.dto.ErrorCode;
import com.spring.basic.chap6.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Service
@Slf4j
@RequiredArgsConstructor
public class MemberService {

    // 의존 객체 설정
    private final MemberRepository memberRepository;

    // 회원 개별 조회 시 비지니스 로직 처리
    public MemberListResponse findOneMember(String account) {  // account 내놔
        log.info("회원 개별 조회를 컨터롤러로부터 위임받음 !");

        // 2. 예외처리 : 계정명이 10글자 초과인지 확인
        if (account.length() > 10) {
            String errorMessage = "계정명이 10글자를 넘어서는 안됩니다.";
            log.warn(errorMessage);
            // 강제 예외를 일으킴 - 코드 흐름이 끊김
            throw new MemberException(ErrorCode.ACCOUNT_TOO_LONG);
        }

        // 1 . 회원 계정명을 통해 데이터 베이스에서 회원정보를 조회해야 함.
        // => DB 접근은 Repository에게 위임
        Member foundMember = memberRepository.findByAccount(account);// 위임

        //               회원이 존재하지 않을 가능성도 처리
        if (foundMember == null) {
            String errorMessage = "회원이 존재하지 않습니다.";
            log.warn(errorMessage);
            throw new MemberException(ErrorCode.MEMBER_NOT_FOUND);
        }

        // 3. 데이터베이스에서 가져온 회원정보를 그대로 응답하면 X - 정제가 필요
        return MemberListResponse.from(foundMember);


    }

    public MemberListResponse createMember(MemberCreateDto dto) {
        Member savedMember = memberRepository.save(MemberCreateDto.from(dto));
        return MemberListResponse.from(savedMember);
    }
}
```

### 서비스 계층의 예외 처리 로직
1. **계정명 길이 검증**: 10글자 초과 시 `ACCOUNT_TOO_LONG` 예외
2. **회원 존재 여부 검증**: 회원이 없으면 `MEMBER_NOT_FOUND` 예외
3. **로깅 처리**: 예외 발생 전 `log.warn()`으로 경고 로그 기록

##  7. 컨트롤러에서의 예외 처리 변화

### MemberApiControllerV6.java
```java
package com.spring.basic.chap6.controller;

import com.spring.basic.chap5_3.dto.request.MemberCreateDto;
import com.spring.basic.chap5_4.dto.response.MemberListResponse;
import com.spring.basic.chap6.service.MemberService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v6/members")
@Slf4j
@RequiredArgsConstructor // final 필드만 골라서 초기화
public class MemberApiControllerV6 {

    // 서비스에게 의존
    private final MemberService memberService;

    /*
        // 생성자 주입     @RequiredArgsConstructor 사용 시 생략가능
        public MemberApiControllerV6(MemberService memberService) {
        this.memberService = memberService;
    }
    */

    // 회원 단일 조회
    @GetMapping("/{account}")
    public ResponseEntity findMember(@PathVariable String account) {
        log.info("회원 단일 조회 요청이 들어옴 ! ");
        log.debug("parameter - account : {}", account);

        // 1 . 회원 계정명을 통해 데이터 베이스에서 회원정보를 조회해야 함.
        // 2. 예외처리 : 계정명이 10글자 미만인지 확인
        //               회원이 존재하지 않을 가능성도 처리
        // 3. 데이터베이스에서 가져온 회원정보를 그대로 응답하면 X - 정제가 필요
        // =======> 서비스에게 위임
//        try {
//            MemberListResponse responseMember = memberService.findOneMember(account);
//            return ResponseEntity.ok(responseMember);
//        } catch (RuntimeException e){
//            return ResponseEntity.badRequest().body(e.getMessage());
//        }
        MemberListResponse responseMember = memberService.findOneMember(account);
        return ResponseEntity.ok(responseMember);

    }

    // 회원 생성
    @PostMapping
    public ResponseEntity create(@RequestBody @Valid MemberCreateDto dto){

        // 서비스에게 위임
        MemberListResponse response = memberService.createMember(dto);

        return ResponseEntity.ok(response);

    }

}
```

### Before vs After 비교

**기존 방식 (주석 처리된 코드)**:
```java
// try {
//     MemberListResponse responseMember = memberService.findOneMember(account);
//     return ResponseEntity.ok(responseMember);
// } catch (RuntimeException e){
//     return ResponseEntity.badRequest().body(e.getMessage());
// }
```

**개선된 방식**:
- **GlobalExceptionHandler가 자동으로 예외를 처리**
- 컨트롤러에서 **try-catch 블록 제거**
- **코드 가독성 향상** 및 **중복 코드 제거**

### 컨트롤러 주요 어노테이션

| 어노테이션 | 설명 |
|-----------|------|
| `@RestController` | REST API 컨트롤러 명시 |
| `@RequestMapping("/api/v6/members")` | 기본 URL 경로 설정 |
| `@RequiredArgsConstructor` | final 필드 생성자 자동 생성 |
| `@GetMapping("/{account}")` | GET 요청 매핑 |
| `@PostMapping` | POST 요청 매핑 |
| `@PathVariable` | URL 경로 변수 바인딩 |
| `@RequestBody @Valid` | 요청 바디 데이터 검증 |

##  8. 예외 처리 흐름도

```
1. 클라이언트 요청 (GET /api/v6/members/{account})
   ↓
2. MemberApiControllerV6.findMember() 호출
   ↓
3. MemberService.findOneMember() 위임
   ↓
4. 비즈니스 로직 실행 (계정명 길이 검증, 회원 존재 확인)
   ↓
5. 예외 상황 발생 시 MemberException 발생
   ↓
6. GlobalExceptionHandler.handleClientException() 자동 호출
   ↓
7. ErrorResponse 객체 생성 및 로깅
   ↓
8. ResponseEntity로 일관성 있는 에러 응답 반환
   ↓
9. 클라이언트에게 JSON 형태의 에러 응답 전달
```

## 9. 핵심 개념 정리

### 스프링 예외처리의 장점
- **관심사의 분리**: 비즈니스 로직과 예외 처리 로직 분리
- **코드 중복 제거**: 여러 컨트롤러에서 공통 예외 처리
- **일관성 있는 응답**: 모든 API에서 동일한 에러 응답 형식
- **유지보수성 향상**: 예외 처리 로직 변경 시 한 곳만 수정

### 구현 시 주의사항
- **적절한 HTTP 상태 코드 사용**: 404(Not Found), 400(Bad Request) 등
- **사용자 친화적인 에러 메시지**: 기술적 내용보다 이해하기 쉬운 메시지
- **로깅 처리**: 디버깅을 위한 적절한 로그 레벨 설정 (`log.warn()`, `log.info()`)
- **보안 고려**: 민감한 정보가 에러 메시지에 노출되지 않도록 주의

### 주요 패턴
1. **ErrorCode enum으로 에러 통합 관리**
2. **사용자 정의 예외 클래스로 구체적인 에러 정보 전달**
3. **@ControllerAdvice로 전역 예외 처리**
4. **Builder 패턴으로 ErrorResponse 객체 생성**

##  추가 학습 포인트

### @Valid와 MethodArgumentNotValidException
- `@RequestBody @Valid`로 입력값 검증
- 검증 실패 시 `MethodArgumentNotValidException` 자동 발생
- GlobalExceptionHandler에서 통합 처리

### ResponseEntity 활용
- HTTP 상태 코드와 응답 바디를 함께 설정
- `ResponseEntity.status(statusCode).body(errorResponse)` 형태로 사용

### 의존성 주입 패턴
- `@RequiredArgsConstructor`로 생성자 주입 자동화
- final 필드만 골라서 생성자 매개변수로 추가
