네, 학습하신 Spring Security, JWT, 예외 처리, DTO 관련 내용을 바탕으로 깔끔한 학습 노트를 만들어 드리겠습니다. 전체 내용을 하나의 마크다운 파일로 정리했으니, 그대로 복사해서 사용하시면 됩니다.

-----

# Spring Security & JWT 기반 로그인/인증 학습 노트


-----

## 1\. 개요: 인증 흐름

이 프로젝트의 인증은 다음과 같은 흐름으로 동작합니다.

1.  **클라이언트 (사용자)**: 회원가입 또는 로그인 정보를 서버에 `Request` (요청)
2.  **Controller (`AuthController`)**: 요청을 받아 유효성을 검증하고 `Service` 계층에 전달
3.  **Service (`UserService`)**: 비즈니스 로직 수행 (ID/PW 검증, 중복 체크 등)
4.  **Repository (`UserRepository`)**: 데이터베이스와 통신하여 사용자 정보 조회/저장
5.  **JWT (`JwtProvider`)**: 인증 성공 시, 사용자 정보를 기반으로 JWT 토큰 생성
6.  **Controller & DTO**: 생성된 토큰과 사용자 정보를 `Response` DTO에 담아 클라이언트에 반환

-----

## 2\. DTO (Data Transfer Object): 계층 간 데이터 전송

DTO는 각 계층(Controller, Service, View) 사이에서 데이터를 주고받기 위해 사용하는 객체입니다. 특히 `Entity`를 직접 노출하지 않고 필요한 데이터만 선택적으로 전달하는 데 중요한 역할을 합니다.

### 요청(Request) DTO

클라이언트의 요청 데이터를 담는 객체입니다. `jakarta.validation` 어노테이션을 사용해 서버에 도달하는 데이터의 유효성을 검증할 수 있습니다.

#### `SignUpRequest.java`

회원가입 요청 시 클라이언트가 보내는 데이터를 담고 검증합니다.

```java
package com.spring.toyproject.domain.dto.request;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;
import lombok.*;

/**
 * 회원 가입 요청에 사용할 DTO
 * - 클라이언트로 부터 회원가입에서 작성한 데이터를 받고 검증하는 객체
 */
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@ToString
public class SignUpRequest {

    // 필드명은 클라이언트가 전송할 때 사용할 JSON의 KEY가 되니 프론트개발자와 협업
    @NotBlank(message = "사용자명은 필수 입니다.")
    @Size(min = 3,max = 15,message = "사용자명은 3자 이상 15자 이하여야 합니다.")
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "사용자명은 영문, 숫자, 언더스코어만 사용 가능합니다.")
    private String username;

    @NotBlank(message = "이메일은 필수입니다.")
    @Email(message = "올바른 이메일 형식이 아닙니다.")
    private String email;

    @NotBlank(message = "비밀번호는 필수입니다.")
    @Size(min = 6, max = 20, message = "비밀번호는 6자 이상 20자 이하여야 합니다.")
    private String password;
}
```

| Validation Annotation | 설명                                                              |
| :-------------------- | :---------------------------------------------------------------- |
| `@NotBlank`           | `null`, `""`, `" "` (공백만 있는 문자열)을 허용하지 않습니다.      |
| `@Size(min, max)`     | 문자열의 길이를 제한합니다.                                       |
| `@Pattern(regexp)`    | 지정된 정규표현식에 맞는 형식이어야 합니다.                       |
| `@Email`              | 이메일 형식(`user@example.com`)이어야 합니다.                     |

#### `LoginRequest.java`

로그인 요청 데이터를 담습니다.

```java
package com.spring.toyproject.domain.dto.request;

import jakarta.validation.constraints.NotBlank;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;

/**
 * 클라이언트가 보낸 로그인 정보를 담는 DTO
 */
@Getter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class LoginRequest {

    @NotBlank(message = "사용자명 또는 이메일은 필수입니다.")
    private String usernameOrEmail;

    @NotBlank(message = "비밀번호는 필수입니다.")
    private String password;

    // 자동로그인 여부 - 요즘은 기본이 자동 로그인임
    // private boolean autoLogin;
}
```

### 응답(Response) DTO

서버가 클라이언트에게 보낼 데이터를 담는 객체입니다.

#### `ApiResponse.java`

모든 API 응답을 일관된 형식으로 래핑(wrapping)하는 공통 응답 객체입니다.

```java
package com.spring.toyproject.domain.dto.common;

import lombok.*;
import java.time.LocalDateTime;

/**
 * 클라이언트에게 일관적인 응답의 포맷을 위한 객체
 */
@EqualsAndHashCode
@Getter @Setter @ToString
@AllArgsConstructor @NoArgsConstructor
@Builder
public class ApiResponse<T> {

    // 응답 성공 여부
    private boolean success;

    // 응답 메시지
    private String message;

    // 응답 시간
    private LocalDateTime timestamp;

    // 실제 응답 데이터 (Generic Type)
    private T data;

    // 정적 팩토리 메서드를 이용해 성공 응답 객체를 쉽게 생성
    public static <T> ApiResponse<T> success(String message, T data) {
        return ApiResponse.<T>builder()
                .success(true)
                .message(message)
                .timestamp(LocalDateTime.now())
                .data(data)
                .build();
    }
}
```

#### `UserResponse.java`

회원가입 완료 또는 사용자 정보 조회 시, 민감 정보(비밀번호 등)를 제외하고 클라이언트에 전달할 사용자 정보를 담습니다.

```java
package com.spring.toyproject.domain.dto.response;

import com.spring.toyproject.domain.entity.User;
import lombok.*;
import java.time.LocalDateTime;

/**
 * 회원가입 직후 또는 마이페이지에서 렌더링에 사용할 JSON 응답 객체
 */
@Getter
@Builder
@ToString
@EqualsAndHashCode
@NoArgsConstructor
@AllArgsConstructor
public class UserResponse {
    private Long id;
    private String username;
    private String email;
    private LocalDateTime createdAt;

    /**
     * User 엔티티를 UserResponseDto로 변환하는 정적 팩토리 메서드
     * Entity의 정보를 그대로 노출하지 않고 DTO로 변환하는 것이 중요
     */
    public static UserResponse from(User user) {
        return UserResponse.builder()
                .id(user.getId())
                .username(user.getUsername())
                .email(user.getEmail())
                .createdAt(user.getCreatedAt())
                .build();
    }
}
```

#### `AuthResponse.java`

로그인 성공 시, JWT 토큰과 사용자 정보를 담아 클라이언트에 전달합니다.

```java
package com.spring.toyproject.domain.dto.response;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;

/**
 * 인증 완료 후 클라이언트에게 전송할 내용
 */
@Getter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class AuthResponse {

    private String token; // JWT 토큰
    @Builder.Default
    private String tokenType = "Bearer"; // 토큰 타입, 'Bearer'는 표준
    private UserResponse user; // 로그인한 유저의 정보

    // 정적 팩토리 메서드
    public static AuthResponse of(String token, UserResponse user) {
        return AuthResponse.builder()
                .token(token)
                .user(user)
                .build();
    }
}
```

-----

## 3\. API 계층 (Controller): 외부 요청 처리

`@RestController`는 클라이언트의 HTTP 요청을 받아 적절한 서비스 메서드를 호출하고, 그 결과를 HTTP 응답으로 반환하는 역할을 합니다.

### `AuthController.java`

회원가입, 로그인 등 인증 관련 API 엔드포인트를 정의합니다.

```java
package com.spring.toyproject.api;

import com.spring.toyproject.domain.dto.common.ApiResponse;
import com.spring.toyproject.domain.dto.request.LoginRequest;
import com.spring.toyproject.domain.dto.request.SignUpRequest;
import com.spring.toyproject.domain.dto.response.AuthResponse;
import com.spring.toyproject.domain.dto.response.UserResponse;
import com.spring.toyproject.service.UserService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/auth") // 이 컨트롤러의 모든 메서드는 /api/auth 로 시작
@Slf4j
@RequiredArgsConstructor
public class AuthController {

    private final UserService userService;

    /**
     * 회원가입 API
     * POST : /api/auth/signup
     */
    @PostMapping("/signup")
    public ResponseEntity<?> signup(@Valid @RequestBody SignUpRequest requestDto) {
        log.info("회원가입 요청: {}", requestDto.getUsername());

        UserResponse response = userService.signup(requestDto);

        // ApiResponse로 감싸서 일관된 형식으로 응답
        return ResponseEntity
                .ok()
                .body(
                        ApiResponse.success("회원가입이 성공적으로 완료되었습니다.", response)
                );
    }

    /**
     * 로그인 API - GET 방식은 URL에 파라미터가 노출될 수 있어 POST 방식 사용
     * POST /api/auth/login
     */
    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody @Valid LoginRequest requestDto) {
        log.info("로그인 요청 : {}", requestDto.getUsernameOrEmail());

        AuthResponse response = userService.authenticate(requestDto);

        return ResponseEntity.ok().body(
                ApiResponse.success("로그인이 완료되었습니다.",response)
        );
    }
}
```

| Annotation           | 설명                                                                                                |
| :------------------- | :-------------------------------------------------------------------------------------------------- |
| `@RestController`    | `@Controller` + `@ResponseBody`. 모든 메서드의 반환 값이 JSON/XML 등 HTTP Body에 직접 쓰여집니다.      |
| `@RequestMapping`    | 클래스나 메서드에 공통 URL 경로를 매핑합니다.                                                       |
| `@PostMapping`       | HTTP POST 요청을 처리하는 메서드에 매핑합니다.                                                      |
| `@RequestBody`       | HTTP 요청의 Body(JSON, XML 등)를 자바 객체(DTO)로 변환합니다.                                        |
| `@Valid`             | DTO 객체의 유효성 검사를 활성화합니다. 검증 실패 시 `MethodArgumentNotValidException`이 발생합니다. |
| `ResponseEntity<?>`  | HTTP 응답의 상태 코드, 헤더, 본문을 포함하는 객체입니다. 유연한 응답 구성이 가능합니다.                |
| `@RequiredArgsConstructor` | `final` 또는 `@NonNull` 필드에 대한 생성자를 자동으로 생성해주는 Lombok 어노테이션입니다. (의존성 주입) |

-----

## 4\. 서비스 계층 (Service): 비즈니스 로직

서비스 계층은 애플리케이션의 핵심 비즈니스 로직을 처리합니다. `@Transactional`을 통해 데이터의 일관성과 무결성을 보장합니다.

### `UserService.java`

회원가입, 로그인 등 사용자 관련 비즈니스 로직을 구현합니다.

```java
package com.spring.toyproject.service;

import com.spring.toyproject.domain.dto.request.LoginRequest;
import com.spring.toyproject.domain.dto.request.SignUpRequest;
import com.spring.toyproject.domain.dto.response.AuthResponse;
import com.spring.toyproject.domain.dto.response.UserResponse;
import com.spring.toyproject.domain.entity.User;
import com.spring.toyproject.exception.BusinessException;
import com.spring.toyproject.exception.ErrorCode;
import com.spring.toyproject.jwt.JwtProvider;
import com.spring.toyproject.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * 사용자 모듈 서비스 클래스
 * 인증, 회원관련 비즈니스 로직 처리
 * 트랜잭션 처리
 */
@Transactional // 메서드 실행 중 예외 발생 시, DB 작업을 롤백
@Service
@RequiredArgsConstructor
@Slf4j
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder; // 비밀번호 암호화를 위한 객체
    private final JwtProvider jwtProvider;         // JWT 토큰을 발급하는 객체

    /**
     * 회원 가입 로직
     */
    public UserResponse signup(SignUpRequest requestDto) {

        // 사용자명 중복 체크
        if (userRepository.existsByUsername(requestDto.getUsername())) {
            throw new BusinessException(ErrorCode.DUPLICATE_USERNAME);
        }
        // 이메일 중복 체크
        if (userRepository.existsByEmail(requestDto.getEmail())) {
            throw new BusinessException(ErrorCode.DUPLICATE_EMAIL);
        }

        // 패스워드를 해시로 암호화
        String encodedPassword = passwordEncoder.encode(requestDto.getPassword());

        // dto를 entity로 변경
        User user = User.builder()
                .email(requestDto.getEmail())
                .username(requestDto.getUsername())
                .password(encodedPassword)
                .build();

        // db에 insert
        User saved = userRepository.save(user);
        log.info("새로운 사용자 가입: {}", saved);

        // Entity를 Response DTO로 변환하여 반환
        return UserResponse.from(saved);
    }

    /**
     * 로그인 로직
     */
    public AuthResponse authenticate(LoginRequest loginRequest) {

        // 사용자 조회 (사용자명 또는 이메일로 조회)
        String inputAccount = loginRequest.getUsernameOrEmail();

        User user = userRepository.findByUsername(inputAccount)
                .orElseGet(() -> userRepository.findByEmail(inputAccount)
                        .orElseThrow(
                                () -> new BusinessException(ErrorCode.USER_NOT_FOUND)
                        )
                );

        // 비밀번호 검증
        String inputPassword = loginRequest.getPassword(); // 사용자가 입력한 평문 패스워드
        String storedPassword = user.getPassword();       // DB에 저장된 암호화된 패스워드

        // 평문을 암호화해서 비교하는 것이 아니라, matches() 메서드로 비교
        if (!passwordEncoder.matches(inputPassword, storedPassword)) {
            throw new BusinessException(ErrorCode.INVALID_PASSWORD);
        }

        // 로그인 성공 시 JWT 토큰 발급
        String token = jwtProvider.generateToken(user.getUsername());
        log.info("사용자 로그인: {}",user.getUsername());

        // 발급 후 -> 클라이언트에게 전송할 DTO 생성
        return AuthResponse.of(token, UserResponse.from(user));
    }
}
```

| Annotation       | 설명                                                                                         |
| :--------------- | :------------------------------------------------------------------------------------------- |
| `@Service`       | 이 클래스가 비즈니스 로직을 담당하는 서비스 계층의 컴포넌트임을 명시합니다.                    |
| `@Transactional` | 클래스 또는 메서드에 트랜잭션을 적용합니다. 작업 중 오류 발생 시 모든 DB 변경사항을 롤백합니다. |

-----

## 5\. 데이터 접근 계층 (Repository & Entity)

데이터베이스의 테이블과 직접적으로 매핑되는 객체(`Entity`)와 해당 `Entity`에 대한 CRUD(Create, Read, Update, Delete) 작업을 수행하는 인터페이스(`Repository`)로 구성됩니다.

### `User.java` (Entity)

`tbl_user` 테이블과 매핑되는 사용자 엔터티입니다.

```java
package com.spring.toyproject.domain.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;
import java.time.LocalDateTime;

/*
    사용자 엔터티
    - 인증 시스템의 핵심 엔터티로 기본 사용자 정보관리
 */
@Entity // 이 클래스가 JPA 엔터티임을 선언
@Table(name = "tbl_user") // 매핑될 테이블 이름 지정
@Getter
@EqualsAndHashCode(of = "id")
@NoArgsConstructor(access = AccessLevel.PROTECTED) // JPA는 기본 생성자가 필요. protected로 안전성 확보
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // DB가 id 자동 생성 및 증가(Auto Increment)
    @Column(name = "user_id")
    private Long id;

    @Column(name = "user_name", unique = true, nullable = false, length = 50)
    private String username;

    @Column(name = "user_email", unique = true, nullable = false, length = 100)
    private String email;

    // 패스워드는 암호화되면 길이가 길어지므로 length를 넉넉하게 설정
    @Column(name = "password", nullable = false, length = 255)
    private String password;

    @CreationTimestamp // 엔터티가 생성될 때의 시간 자동 저장
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp // 엔터티가 수정될 때의 시간 자동 저장
    @Column(nullable = false)
    private LocalDateTime updatedAt;

    @Builder // 빌더 패턴으로 객체 생성
    public User(String username, String email, String password) {
        this.username = username;
        this.email = email;
        this.password = password;
    }
}
```

### `UserRepository.java` (Repository)

Spring Data JPA를 사용하여 `User` 엔터티에 대한 데이터베이스 작업을 처리합니다. `JpaRepository`를 상속받으면 기본적인 CRUD 메서드(e.g., `save()`, `findById()`, `findAll()`)가 자동으로 제공됩니다.

```java
package com.spring.toyproject.repository;

import com.spring.toyproject.domain.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {

    /**
     * 로그인 시 이름이나 이메일로 사용자를 조회해야 함
     * Optional<T>는 결과가 null일 수 있음을 명시적으로 표현하여 NullPointerException을 방지
     */
    Optional<User> findByUsername(String username);
    Optional<User> findByEmail(String email);

    // 회원가입 시 중복 여부 확인
    boolean existsByUsername(String username);
    boolean existsByEmail(String email);
}
```

**쿼리 메서드 (Query Method)**: Spring Data JPA는 메서드 이름을 분석하여 자동으로 SQL 쿼리를 생성합니다.

* `findByUsername(String username)` → `SELECT * FROM tbl_user WHERE user_name = ?`
* `existsByEmail(String email)` → `SELECT CASE WHEN COUNT(*) > 0 THEN TRUE ELSE FALSE END FROM tbl_user WHERE user_email = ?`

-----

## 6\. JWT(JSON Web Token): 개념 및 구현

### JWT란?

JSON 객체를 사용하여 정보를 안전하게 전송하기 위한 컴팩트하고 독립적인 방법을 정의한 개방형 표준(RFC 7519)입니다. 주로 아래 두 가지 목적으로 사용됩니다.

1.  **인증 (Authentication)**: 사용자가 로그인하면 서버는 JWT를 발급합니다. 이후 사용자는 보호된 라우트나 리소스에 접근할 때마다 이 토큰을 제시하여 자신을 증명합니다.
2.  **정보 교환 (Information Exchange)**: 두 당사자 간에 정보를 안전하게 전송하는 데 사용됩니다. 토큰이 서명되어 있으므로 정보가 변조되지 않았는지 신뢰할 수 있습니다.

### JWT 구조

JWT는 `.`을 구분자로 세 부분으로 나뉩니다.

`xxxxx.yyyyy.zzzzz`

1.  **헤더 (Header)**: 토큰의 타입(JWT)과 사용된 서명 알고리즘(e.g., HMAC SHA256, RS256)을 지정합니다.
2.  **페이로드 (Payload)**: 클레임(Claim)을 포함합니다. 클레임은 사용자에 대한 정보(e.g., 사용자 ID, 이름)나 토큰에 대한 추가 데이터(e.g., 만료 시간 `exp`)를 담고 있습니다.
3.  **서명 (Signature)**: 인코딩된 헤더, 인코딩된 페이로드, 그리고 \*\*비밀 키(Secret Key)\*\*를 사용하여 생성됩니다. 이 서명은 메시지가 도중에 변경되지 않았음을 확인하는 데 사용됩니다. **서버만이 비밀 키를 알고 있으므로, 서명을 검증함으로써 토큰의 신뢰성을 확인할 수 있습니다.**

### JWT 구현 클래스

#### `JwtProperties.java`

`application.yml` 파일에 정의된 JWT 관련 설정 값(비밀 키, 만료 시간)을 읽어오는 클래스입니다.

```java
package com.spring.toyproject.jwt;

import lombok.Getter;
import lombok.Setter;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

/**
 * JWT 설정 읽기 파일 - application.yml 의 jwt.xx 값을 읽어오는 클래스
 */
@Setter @Getter
@Configuration
@ConfigurationProperties(prefix = "jwt") // "jwt" 접두사를 가진 프로퍼티를 매핑
public class JwtProperties {

    private String secret; // jwt.secret 값을 읽어들임
    private Long expiration; // jwt.expiration 값을 읽어들임
}
```

#### `JwtProvider.java`

JWT 토큰을 생성, 검증, 파싱하는 핵심 유틸리티 클래스입니다.

```java
package com.spring.toyproject.jwt;

import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;

/**
 * JWT 토큰 생성, 검증, 파싱 기능 제공 유틸클래스
 */
@Component
@Slf4j
@RequiredArgsConstructor
public class JwtProvider {

    // jwt 설정값 읽기 클래스 주입
    private final JwtProperties jwtProperties;

    /**
     * JWT 토큰을 발급하는 메서드
     * @param username - 발급 대상의 사용자 이름 (토큰의 주체, subject)
     * @return - 암호화된 JWT 토큰 문자열
     */
    public String generateToken(String username) {

        Date now = new Date();
        Date expiryDate  = new Date(now.getTime() + jwtProperties.getExpiration());

        return Jwts.builder()
                .subject(username)       // sub: 토큰의 주체 (누구의 토큰인가)
                .issuedAt(now)           // iat: 토큰 발급 시간
                .expiration(expiryDate)  // exp: 토큰 만료 시간
                .issuer("Toy Project")   // iss: 토큰 발급자
                .signWith(getSigningKey()) // 서명
                .compact(); // 문자열로 압축
    }

    /**
     * JWT 토큰 발급에 필요한 서명 키 생성
     * @return 서명 키 객체 (SecretKey)
     */
    private SecretKey getSigningKey(){
        // application.yml의 secret 값을 UTF-8 바이트 배열로 변환하여 HMAC-SHA 키 생성
        return Keys.hmacShaKeyFor(jwtProperties.getSecret().getBytes(StandardCharsets.UTF_8));
    }
}
```

-----

## 7\. Spring Security 설정

Spring Security는 강력한 인증 및 인가 프레임워크입니다. JWT 기반의 `Stateless`(무상태) 인증을 위해 기본 설정을 변경해야 합니다.

### `PasswordEncoderConfig.java`

비밀번호를 안전하게 저장하기 위해 암호화(해싱)하는 `PasswordEncoder`를 Bean으로 등록합니다. `BCryptPasswordEncoder`는 강력한 해시 함수로, 현재 널리 사용되는 표준입니다.

```java
package com.spring.toyproject.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

/**
 * 비밀번호나 민감정보 해시 암호화를 위한 빈 등록
 */
@Configuration // 이 클래스가 스프링의 설정 정보를 포함함을 명시
public class PasswordEncoderConfig {

    @Bean // 이 메서드가 반환하는 객체를 스프링 컨테이너가 관리하는 Bean으로 등록
    public PasswordEncoder passwordEncoder(){
        // BCrypt 해시 알고리즘을 사용하는 인코더를 반환
        return new BCryptPasswordEncoder();
    }
}
```

### `SecurityConfig.java`

애플리케이션의 전반적인 보안 규칙을 설정합니다.

```java
package com.spring.toyproject.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

/**
 * Spring Security 설정 클래스
 * 인증 설정 및 기본 보안 규칙 설정
 */
@Configuration
@EnableWebSecurity // 스프링 시큐리티 필터 체인을 활성화
public class SecurityConfig {

    // 기본 인증 옵션 설정
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {

        http
            // CSRF(Cross-Site Request Forgery) 공격 방지 기능 비활성화
            // JWT는 서버에 인증 정보를 저장하지 않으므로 CSRF에 비교적 안전
            .csrf(AbstractHttpConfigurer::disable)
            // CORS(Cross-Origin Resource Sharing) 설정은 나중에 수동으로 구성
            .cors(cors -> cors.configure(http))
            // 세션 관리 정책을 STATELESS로 설정. 즉, 서버가 세션을 생성하거나 사용하지 않음
            .sessionManagement(session ->
                    session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            // Spring Security가 기본으로 제공하는 로그인 폼 비활성화
            .formLogin(AbstractHttpConfigurer::disable)
            // HTTP Basic 인증 방식 비활성화
            .httpBasic(AbstractHttpConfigurer::disable)
        ;

        // 나중에 JWT 필터를 추가할 예정 (예: .addFilterBefore(...))
        // 현재는 모든 요청을 허용하도록 설정할 수 있음
        // http.authorizeHttpRequests(auth -> auth.anyRequest().permitAll());

        return http.build();
    }
}
```

-----

## 8\. 전역 예외 처리 (Global Exception Handling)

애플리케이션 전반에 걸쳐 발생하는 예외를 한 곳에서 일관된 방식으로 처리하기 위한 메커니즘입니다. `@RestControllerAdvice`를 사용해 구현합니다.

### 예외 클래스

#### `ErrorCode.java`

애플리케이션에서 발생할 수 있는 모든 에러의 종류(코드, 메시지, 상태)를 열거형(Enum)으로 정의하여 체계적으로 관리합니다.

```java
package com.spring.toyproject.exception;

import lombok.AllArgsConstructor;
import lombok.Getter;

/**
 * 애플리케이션에서 발생할 수 있는 에러 코드들을 정의하는 열거체
 * 에러상태코드, 에러메시지, 에러이름을 함께 관리
 */
@Getter
@AllArgsConstructor
public enum ErrorCode {

    // ... (중략) ...

    // 인증 관련 에러 코드
    USER_NOT_FOUND("USER_NOT_FOUND", "사용자를 찾을 수 없습니다.", 404),
    DUPLICATE_USERNAME("DUPLICATE_USERNAME", "이미 사용 중인 사용자명입니다.", 409),
    DUPLICATE_EMAIL("DUPLICATE_EMAIL", "이미 사용 중인 이메일입니다.", 409),
    INVALID_PASSWORD("INVALID_PASSWORD", "비밀번호가 올바르지 않습니다.", 401),
    
    // 유효성 검증 에러
    VALIDATION_ERROR("VALIDATION_ERROR", "유효성 검사에 실패했습니다.", 400);

    // ... (나머지 코드) ...

    private final String code;
    private final String message;
    private final int status;
}
```

#### `BusinessException.java`

서비스 로직 수행 중 발생하는 예외를 나타내는 커스텀 예외 클래스입니다. `ErrorCode`를 포함하여 어떤 종류의 비즈니스 예외인지 명확히 합니다.

```java
package com.spring.toyproject.exception;

import lombok.Getter;
import lombok.NoArgsConstructor;

/**
 * 범용적인 에러가 아니라 우리 앱에서만 발생하는 독특한 에러들을 저장하는 예외클래스
 */
@Getter
@NoArgsConstructor
public class BusinessException extends RuntimeException {

    private ErrorCode errorCode;

    public BusinessException(String message) {
        super(message);
    }
    
    // ErrorCode를 받아 에러 메시지와 코드를 설정
    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }
}
```

### 예외 응답 DTO

#### `ErrorResponse.java`

예외 발생 시 클라이언트에게 반환할 응답의 형식을 정의한 DTO입니다.

```java
package com.spring.toyproject.exception.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import java.time.LocalDateTime;
import java.util.List;

@Getter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL) // 값이 null인 프로퍼티는 응답 JSON에서 제외
public class ErrorResponse {

    private LocalDateTime timestamp;
    private int status; // HTTP 상태 코드 (e.g., 404, 500)
    private String error; // 에러 코드 이름 (e.g., USER_NOT_FOUND)
    private String detail; // 상세 에러 메시지
    private String path; // 에러가 발생한 요청 URI
    private List<ValidationError> validationErrors; // 유효성 검증 에러 목록

    // 입력값 검증 오류 1개를 포장할 내부 클래스
    @Getter
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class ValidationError {
        private String field; // 에러가 발생한 필드명
        private String message; // 에러 원인 메시지
        private Object rejectedValue; // 클라이언트가 보낸 거부된 값
    }
}
```

### 전역 예외 핸들러

#### `GlobalExceptionHandler.java`

`@RestControllerAdvice`를 사용하여 모든 컨트롤러에서 발생하는 예외를 가로채 처리합니다. `@ExceptionHandler`를 통해 특정 예외 타입별로 처리 로직을 분리할 수 있습니다.

```java
package com.spring.toyproject.exception;

import com.spring.toyproject.exception.dto.ErrorResponse;
import jakarta.servlet.http.HttpServletRequest;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import java.time.LocalDateTime;
import java.util.List;
import java.util.stream.Collectors;

/**
 * 전역 예외처리 핸들러
 * 애플리케이션에서 발생하는 모든 예외를 일관된 형식으로 처리
 */
@Slf4j
@RestControllerAdvice // 모든 @RestController에 대한 AOP(관점 지향 프로그래밍) 적용
public class GlobalExceptionHandler {

    /**
     * 우리 앱에서 직접 정의한 BusinessException을 처리
     * @param e 발생한 예외 객체
     * @param request 예외가 발생한 HTTP 요청 정보
     * @return ErrorResponse를 담은 ResponseEntity
     */
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<?> handleBusinessException(BusinessException e, HttpServletRequest request) {
        log.warn("비즈니스 예외 발생 : {}", e.getMessage());

        // 에러 응답 객체 생성
        ErrorResponse errorResponse = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .detail(e.getMessage())
                .path(request.getRequestURI())
                .status(e.getErrorCode().getStatus())
                .error(e.getErrorCode().getCode())
                .build();

        return ResponseEntity.status(e.getErrorCode().getStatus()).body(errorResponse);
    }

    /**
     * DTO 유효성 검증(@Valid) 실패 시 발생하는 예외 처리
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(MethodArgumentNotValidException e) {
        // 어떤 필드에서 어떤 에러가 발생했는지 상세 정보를 추출
        List<ErrorResponse.ValidationError> validationErrors = e.getBindingResult()
                .getFieldErrors()
                .stream()
                .map(fieldError -> ErrorResponse.ValidationError.builder()
                        .field(fieldError.getField())
                        .message(fieldError.getDefaultMessage())
                        .rejectedValue(fieldError.getRejectedValue())
                        .build())
                .collect(Collectors.toList());

        log.warn("유효성 검증 실패: {}", validationErrors);

        ErrorResponse response = ErrorResponse.builder()
                .validationErrors(validationErrors)
                .timestamp(LocalDateTime.now())
                .error(ErrorCode.VALIDATION_ERROR.getCode())
                .status(ErrorCode.VALIDATION_ERROR.getStatus())
                .message(ErrorCode.VALIDATION_ERROR.getMessage()) // 추가
                .build();

        return ResponseEntity.badRequest().body(response);
    }
}
```

| Annotation           | 설명                                                                                             |
| :------------------- | :----------------------------------------------------------------------------------------------- |
| `@RestControllerAdvice` | 전역적으로 컨트롤러에 적용되는 `@ExceptionHandler`, `@InitBinder`, `@ModelAttribute`를 정의합니다. |
| `@ExceptionHandler`  | 특정 예외 클래스가 발생했을 때 실행될 메서드를 지정합니다.                                       |