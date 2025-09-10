네, 요청하신 `쿠키 설정`과 `카카오 인증` 학습 내용을 바탕으로 하나의 마크다운(MD) 파일을 만들었습니다. 아래 내용을 복사하여 사용하시면 됩니다.

````markdown
# AI Study Mate - 백엔드 인증 및 소셜 로그인 학습 노트

## 1. 프로젝트 개요 및 설정

이 노트는 Spring Boot와 React를 사용하여 AI 스터디 메이트 프로젝트의 인증 시스템을 구축하는 과정을 기록합니다. 주요 학습 목표는 HTTP-Only 쿠키를 이용한 JWT 인증과 OAuth2를 활용한 카카오 소셜 로그인 구현입니다.

### 1.1. Backend 설정 (`build.gradle`)

프로젝트에 필요한 주요 라이브러리 의존성입니다.

```groovy
dependencies {
    // Spring Web & Data JPA
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'

    // Security & OAuth2 for authentication
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'

    // JWT for token-based authentication
    implementation 'io.jsonwebtoken:jjwt-api:0.12.3'
    runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.3'
    runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.3'

    // Dotenv for managing environment variables
    implementation 'me.paulschwarz:spring-dotenv:4.0.0'

    // QueryDSL for type-safe queries
    implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
    annotationProcessor 'com.querydsl:querydsl-apt:5.0.0:jakarta'
    annotationProcessor 'jakarta.persistence:jakarta.persistence-api:3.1.0'

    // H2 Database for development
    runtimeOnly 'com.h2database:h2'

    // Lombok for reducing boilerplate code
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
}
````

### 1.2. Frontend 설정

#### 1.2.1. API 클라이언트 (`apiClient.js`)

Axios를 사용하여 백엔드와 통신하는 클라이언트를 설정합니다.

- `baseURL: '/api'`: `/api`로 시작하는 모든 요청을 Vite 프록시를 통해 백엔드 서버(`http://localhost:9005`)로 전달합니다.
- `withCredentials: true`: 프론트엔드와 백엔드 간에 쿠키(특히 HTTP-Only 쿠키)를 주고받을 수 있도록 설정합니다.

<!-- end list -->

```javascript
// frontend/src/services/apiClient.js
import axios from 'axios';

// Axios 기본 설정
const apiClient = axios.create({
  baseURL: '/api', // Vite 프록시 사용
  withCredentials: true, // HTTP-Only 쿠키 지원
});

export default apiClient;
```

#### 1.2.2. Vite 프록시 설정 (`vite.config.js`)

개발 중 발생하는 CORS(Cross-Origin Resource Sharing) 문제를 해결하기 위해 Vite의 프록시 기능을 사용합니다. 프론트엔드(`localhost:3000`)에서 `/api` 경로로 보내는 요청을 백엔드(`localhost:9005`)로 전달합니다.

```javascript
// frontend/vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:9005', // 백엔드 서버 주소
        changeOrigin: true, // Origin 헤더를 백엔드 주소로 변경
        rewrite: (path) => path.replace(/^\/api/, ''), // '/api' 접두사 제거
      },
    },
  },
});
```

-----

## 2\. JWT 및 HTTP-Only 쿠키를 이용한 인증

세션 대신 JWT(JSON Web Token)와 HTTP-Only 쿠키를 사용하여 Stateless 인증 시스템을 구축합니다.

### 2.1. 환경 변수 설정 (`application.yml`)

`application.yml` 파일에서 JWT와 쿠키 관련 설정을 관리합니다. 민감한 정보는 `${...}` 플레이스홀더를 통해 환경 변수에서 주입받습니다.

#### **`application-dev.yml` (개발 환경)**

개발 환경에서는 `localhost` 도메인과 `http` 통신을 가정하여 `secure: false`, `sameSite: Lax`로 설정합니다.

```yaml
# backend/src/main/resources/application-dev.yml
cookie:
  domain: localhost
  secure: false
  http-only: true
  same-site: Lax # 개발 환경에서는 Lax로 설정
  access-token-name: ACCESS_TOKEN
  refresh-token-name: REFRESH_TOKEN
  access-max-age: 900 # 15분
  refresh-max-age: 604800 # 7일

jwt:
  secret: ${JWT_SECRET}
  access-expiration: 900000 # 15분
  refresh-expiration: 604800000 # 7일
```

#### **`application-prod.yml` (운영 환경)**

운영 환경에서는 HTTPS를 사용하므로 `secure: true`로, 크로스-도메인 요청을 위해 `sameSite: None`으로 설정합니다.

```yaml
# backend/src/main/resources/application-prod.yml
cookie:
  domain: ${COOKIE_DOMAIN}
  secure: true
  http-only: true
  same-site: None # SameSite=None을 사용하려면 secure=true가 필수
  access-token-name: ACCESS_TOKEN
  refresh-token-name: REFRESH_TOKEN
  access-max-age: 900
  refresh-max-age: 604800
```

### 2.2. 설정 프로퍼티 클래스

`@ConfigurationProperties` 어노테이션을 사용하여 `application.yml`의 설정 값들을 자바 객체로 바인딩합니다.

#### **`CookieProperties.java`**

```java
// backend/src/main/java/com/study/mate/util/CookieProperties.java
package com.study.mate.util;

import lombok.Getter;
import lombok.Setter;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Getter
@Setter
@Configuration
@ConfigurationProperties(prefix = "cookie") // "cookie" 접두사를 가진 프로퍼티를 매핑
public class CookieProperties {
    private String domain;
    private boolean secure;
    private boolean httpOnly;
    private String sameSite; // Lax, Strict, None
    private String accessTokenName;
    private String refreshTokenName;
    private int accessMaxAge; // seconds
    private int refreshMaxAge; // seconds
}
```

#### **`JwtProperties.java`**

```java
// backend/src/main/java/com/study/mate/util/JwtProperties.java
package com.study.mate.util;

import lombok.Getter;
import lombok.Setter;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Getter
@Setter
@Configuration
@ConfigurationProperties(prefix = "jwt") // "jwt" 접두사를 가진 프로퍼티를 매핑
public class JwtProperties {
    private String secret;
    private long accessExpiration;
    private long refreshExpiration;
}
```

### 2.3. 쿠키 유틸리티 (`CookieUtil.java`)

`ResponseCookie`를 사용하여 일관된 정책(HttpOnly, Secure, SameSite 등)으로 쿠키를 생성하고 삭제하는 유틸리티 클래스입니다.

```java
// backend/src/main/java/com/study/mate/util/CookieUtil.java
package com.study.mate.util;

import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseCookie;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
public class CookieUtil {

    private final CookieProperties props;

    // Access Token 쿠키 추가
    public void addAccessTokenCookie(HttpServletResponse response, String token) {
        ResponseCookie cookie = buildCookie(props.getAccessTokenName(), token, props.getAccessMaxAge());
        setCookieHeader(response, cookie);
    }

    // Refresh Token 쿠키 추가
    public void addRefreshTokenCookie(HttpServletResponse response, String token) {
        ResponseCookie cookie = buildCookie(props.getRefreshTokenName(), token, props.getRefreshMaxAge());
        setCookieHeader(response, cookie);
    }

    // 쿠키 삭제 (max-age=0)
    public void deleteAccessTokenCookie(HttpServletResponse response) {
        deleteCookie(response, props.getAccessTokenName());
    }

    // 공통 쿠키 빌더
    private ResponseCookie buildCookie(String name, String value, int maxAgeSeconds) {
        return ResponseCookie.from(name, value)
                .httpOnly(props.isHttpOnly())
                .secure(props.isSecure())
                .domain(props.getDomain())
                .path("/")
                .sameSite(props.getSameSite())
                .maxAge(maxAgeSeconds)
                .build();
    }
    
    // Set-Cookie 헤더에 쿠키 추가
    private void setCookieHeader(HttpServletResponse response, ResponseCookie cookie) {
        response.addHeader("Set-Cookie", cookie.toString());
    }
}
```

### 2.4. JWT 토큰 제공자 (`JwtTokenProvider.java`)

JWT를 생성하고, 서명을 검증하며, 토큰에서 정보를 추출하는 핵심 로직을 담당합니다.

- `init()`: `@PostConstruct`를 사용하여 애플리케이션 시작 시 Base64로 인코딩된 시크릿 키를 `SecretKey` 객체로 변환합니다.
- `createAccessToken()`, `createRefreshToken()`: 각각의 용도에 맞는 토큰을 생성합니다.
- `validateToken()`, `getClaims()`: 토큰의 유효성을 검증하고 토큰에 담긴 정보를 추출합니다.

<!-- end list -->

```java
// backend/src/main/java/com/study/mate/util/JwtTokenProvider.java
package com.study.mate.util;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;
import jakarta.annotation.PostConstruct;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import javax.crypto.SecretKey;
import java.util.Date;
import java.util.Map;

@Component
@RequiredArgsConstructor
public class JwtTokenProvider {

    private final JwtProperties jwtProperties;
    private SecretKey key;

    // 애플리케이션 시작 시 시크릿 키 초기화
    @PostConstruct
    public void init() {
        this.key = toSecretKey(jwtProperties.getSecret());
    }

    // Base64 시크릿을 HMAC-SHA 서명 키로 변환
    private SecretKey toSecretKey(String base64Secret) {
        return Keys.hmacShaKeyFor(Decoders.BASE64.decode(base64Secret));
    }

    // 공통 토큰 생성 헬퍼
    private String createToken(String subject, Map<String, Object> claims, long expirationMillis) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + expirationMillis);
        return Jwts.builder()
                .subject(subject)
                .claims(claims)
                .issuedAt(now)
                .expiration(expiry)
                .signWith(key)
                .compact();
    }

    // 서명 검증 후 Claims 파싱
    private Claims parseClaims(String token) {
        return Jwts.parser().verifyWith(key).build().parseSignedClaims(token).getPayload();
    }

    // Access Token 생성
    public String createAccessToken(String subject, Map<String, Object> claims) {
        return createToken(subject, claims, jwtProperties.getAccessExpiration());
    }

    // Refresh Token 생성
    public String createRefreshToken(String subject) {
        return createToken(subject, Map.of(), jwtProperties.getRefreshExpiration());
    }

    // 토큰 유효성 검증
    public boolean validateToken(String token) {
        try {
            parseClaims(token);
            return true;
        } catch (Exception e) {
            return false;
        }
    }

    // 토큰에서 Claims 추출
    public Claims getClaims(String token) {
        return parseClaims(token);
    }
}
```

| 메서드 | 설명 | 파라미터 | 반환값 |
| --- | --- | --- | --- |
| `createAccessToken` | Access Token을 생성합니다. | `subject`: 사용자 식별자, `claims`: 추가 정보 | `String` (JWT) |
| `createRefreshToken` | Refresh Token을 생성합니다. | `subject`: 사용자 식별자 | `String` (JWT) |
| `validateToken` | 토큰의 서명과 만료 시간을 검증합니다. | `token`: JWT | `boolean` |
| `getClaims` | 유효한 토큰에서 Claim(정보)을 추출합니다. | `token`: JWT | `Claims` |

-----

## 3\. Spring Security 설정

Spring Security를 사용하여 애플리케이션의 보안을 관리합니다.

### 3.1. 초기 Security 설정 (쿠키 실습)

초기 단계에서는 CORS 설정과 기본적인 경로 접근 권한만 설정합니다.

```java
// 쿠키 브랜치의 SecurityConfig.java
package com.study.mate.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.Arrays;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource())) // CORS 설정
            .authorizeHttpRequests(authz -> authz
                // '/health', '/', '/h2-console/**' 경로는 인증 없이 허용
                .requestMatchers("/health", "/", "/h2-console/**").permitAll()
                .anyRequest().authenticated() // 나머지 요청은 인증 필요
            )
            .csrf(csrf -> csrf
                .ignoringRequestMatchers("/h2-console/**") // H2 콘솔은 CSRF 보호 예외
            )
            .headers(headers -> headers
                .frameOptions(frame -> frame.sameOrigin()) // H2 콘솔 iframe 허용
            );
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        // ... (CORS 설정 로직)
    }
}
```

### 3.2. JWT 인증 필터 추가 (`JwtAuthenticationFilter.java`)

모든 요청이 컨트롤러에 도달하기 전에 JWT를 검증하는 커스텀 필터입니다. `OncePerRequestFilter`를 상속하여 요청당 한 번만 실행되도록 보장합니다.

- **`doFilterInternal`**: 요청 헤더나 쿠키에서 토큰을 추출하고, 유효성을 검증한 뒤 `SecurityContextHolder`에 인증 정보를 저장합니다.
- **`resolveToken`**: `Authorization` 헤더의 Bearer 토큰을 우선적으로 찾고, 없으면 쿠키에서 Access Token을 찾습니다.

<!-- end list -->

```java
// backend/src/main/java/com/study/mate/config/JwtAuthenticationFilter.java
package com.study.mate.config;

import com.study.mate.util.CookieProperties;
import com.study.mate.util.JwtTokenProvider;
import io.jsonwebtoken.Claims;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.Cookie;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.lang.NonNull;
import lombok.RequiredArgsConstructor;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.Collections;

@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;
    private final CookieProperties cookieProperties;

    @Override
    protected void doFilterInternal(@NonNull HttpServletRequest request, @NonNull HttpServletResponse response, @NonNull FilterChain filterChain)
            throws ServletException, IOException {

        String token = resolveToken(request);

        // 토큰이 유효하고, SecurityContext에 인증 정보가 없을 경우
        if (StringUtils.hasText(token) && jwtTokenProvider.validateToken(token)
                && SecurityContextHolder.getContext().getAuthentication() == null) {
            Claims claims = jwtTokenProvider.getClaims(token);
            String subject = claims.getSubject(); // 사용자 식별자

            // 인증 객체 생성
            Authentication authentication = new UsernamePasswordAuthenticationToken(
                    subject, null, Collections.emptyList());
            
            // SecurityContext에 인증 정보 저장
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);
    }

    // 요청에서 Access Token 추출
    private String resolveToken(HttpServletRequest request) {
        // 1. Authorization 헤더에서 Bearer 토큰 추출
        String bearer = request.getHeader("Authorization");
        if (StringUtils.hasText(bearer) && bearer.startsWith("Bearer ")) {
            return bearer.substring(7);
        }

        // 2. 쿠키에서 Access Token 추출
        Cookie[] cookies = request.getCookies();
        if (cookies != null) {
            for (Cookie cookie : cookies) {
                if (cookieProperties.getAccessTokenName().equals(cookie.getName())) {
                    return cookie.getValue();
                }
            }
        }
        return null;
    }
}
```

### 3.3. 최종 Security 설정 (JWT 필터 적용)

`JwtAuthenticationFilter`를 Security Filter Chain에 추가하고, 세션 관리 방식을 `STATELESS`로 변경하여 완전한 토큰 기반 인증을 구현합니다.

```java
// 카카오 로그인 브랜치의 SecurityConfig.java
package com.study.mate.config;

// ... imports
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http, JwtAuthenticationFilter jwtAuthenticationFilter) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable()) // JWT 기반이므로 CSRF 비활성화
            // 세션 관리 방식을 STATELESS로 설정
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .formLogin(form -> form.disable())
            .httpBasic(basic -> basic.disable())
            .headers(headers -> headers.frameOptions(frame -> frame.sameOrigin()))
            .authorizeHttpRequests(authz -> authz
                // 인증 없이 접근 가능한 Public 경로들
                .requestMatchers("/health", "/", "/h2-console/**", "/oauth2/**", "/login/oauth2/**").permitAll()
                .anyRequest().authenticated()
            )
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((req, res, e) -> res.sendError(401)) // 인증 실패 시 401 Unauthorized
            )
            // UsernamePasswordAuthenticationFilter 앞에 커스텀 JWT 필터 추가
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
    // ...
}
```

-----

## 4\. 카카오 소셜 로그인 구현

### 4.1. 카카오 로그인 흐름

1.  **사용자**: 프론트엔드에서 "카카오로 로그인" 버튼 클릭.
2.  **프론트엔드 -\> 백엔드**: `GET /oauth2/kakao` 요청.
3.  **백엔드**: 카카오 인증 서버로 리다이렉트 응답.
    - 이때 `client_id`, `redirect_uri` 등의 파라미터를 포함.
4.  **사용자 (브라우저)**: 카카오 로그인 페이지로 이동하여 로그인 및 동의.
5.  **카카오 -\> 백엔드**: 사용자가 동의하면, 카카오가 지정된 `redirect_uri`(`GET /login/oauth2/code/kakao`)로 \*\*인가 코드(Authorization Code)\*\*와 함께 리다이렉트.
6.  **백엔드**: 받은 인가 코드를 사용하여 카카오에 **Access Token**을 요청 (Server-to-Server 통신).
7.  **백엔드**: 받은 Access Token으로 카카오에 **사용자 정보**를 요청.
8.  **백엔드**:
    - 받아온 사용자 정보(이메일, 닉네임 등)로 DB에서 사용자를 조회하거나 신규 가입 처리.
    - 자체 서비스의 **JWT(Access Token, Refresh Token)를 생성**.
    - 생성된 토큰을 HTTP-Only 쿠키에 담아 클라이언트에 응답.

### 4.2. 카카오 로그인 관련 설정

`application.yml`에 카카오 애플리케이션의 키와 리다이렉트 URI를 설정합니다.

```yaml
# backend/src/main/resources/application.yml
sns:
  appKey: ${KAKAO_CLIENT_ID}
  clientSecret: ${KAKAO_CLIENT_SECRET}
  redirect: http://localhost:9005/login/oauth2/code/kakao
```

### 4.3. Controller (`SnsController.java`)

카카오 로그인 요청을 받고, 인가 코드를 받아 서비스 로직을 호출하는 컨트롤러입니다.

```java
// backend/src/main/java/com/study/mate/controller/SnsController.java
package com.study.mate.controller;

import com.study.mate.service.SnsService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import java.util.Map;

@Controller
@Slf4j
@RequiredArgsConstructor
public class SnsController {

    private final SnsService service;

    @Value("${sns.appKey}")
    private String appKey;
    @Value("${sns.redirect}")
    private String redirectUri;
    @Value("${sns.clientSecret}")
    private String clientSecret;

    // 1. 카카오 인가 코드 발급 요청 (클라이언트 -> 백엔드)
    @GetMapping("/oauth2/kakao")
    public String kakao() {
        String uri = "[https://kauth.kakao.com/oauth/authorize](https://kauth.kakao.com/oauth/authorize)"
            + "?client_id=" + appKey
            + "&redirect_uri=" + redirectUri
            + "&response_type=code";
        // 카카오 인증 페이지로 리다이렉트
        return "redirect:" + uri;
    }

    // 2. 인가 코드를 받아 토큰 발급 및 사용자 정보 처리 (카카오 -> 백엔드)
    @GetMapping("/login/oauth2/code/kakao")
    public String kakaoCode(@RequestParam String code) {
        log.info("카카오가 전달한 인가코드 : {}", code);

        service.kakaoLogin(Map.of(
                "client_id", appKey,
                "redirect_uri", redirectUri,
                "code", code,
                "client_secret", clientSecret
        ));

        // 로그인 성공 후 프론트엔드의 특정 페이지로 리다이렉트 (예: 홈)
        return "redirect:/";
    }
}
```

### 4.4. Service (`SnsService.java`)

`RestTemplate`을 사용하여 백엔드 서버가 직접 카카오 API 서버와 통신하며 실제 로직을 처리합니다.

```java
// backend/src/main/java/com/study/mate/service/SnsService.java
package com.study.mate.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.*;
import org.springframework.stereotype.Service;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.web.client.RestTemplate;
import java.util.Map;

@Service
@Slf4j
@RequiredArgsConstructor
public class SnsService {

    public void kakaoLogin(Map<String, Object> params) {
        // 1. 인가 코드로 토큰 발급 요청
        String accessToken = getKakaoToken(params);

        // 2. 발급한 토큰으로 사용자 정보 가져오기
        getKakaoUserInfo(accessToken);
    }
    
    // 인가코드로 토큰 발급 요청
    private String getKakaoToken(Map<String, Object> params) {
        String uri = "[https://kauth.kakao.com/oauth/token](https://kauth.kakao.com/oauth/token)";
        HttpHeaders headers = new HttpHeaders();
        headers.add("Content-Type", "application/x-www-form-urlencoded;charset=utf-8");

        LinkedMultiValueMap<String, Object> valueMap = new LinkedMultiValueMap<>();
        valueMap.add("grant_type", "authorization_code");
        valueMap.add("client_id", params.get("client_id"));
        valueMap.add("redirect_uri", params.get("redirect_uri"));
        valueMap.add("code", params.get("code"));
        valueMap.add("client_secret", params.get("client_secret"));

        HttpEntity<Object> entity = new HttpEntity<>(valueMap, headers);
        RestTemplate template = new RestTemplate();
        
        // 서버-서버 통신으로 POST 요청 보내기
        ResponseEntity<Map> response = template.exchange(uri, HttpMethod.POST, entity, Map.class);
        Map<String, Object> responseBody = response.getBody();
        String accessToken = (String) responseBody.get("access_token");
        log.info("카카오 Access Token 발급 성공: {}", accessToken);
        return accessToken;
    }

    // 토큰으로 사용자 정보 조회
    private void getKakaoUserInfo(String accessToken) {
        String requestUri = "[https://kapi.kakao.com/v2/user/me](https://kapi.kakao.com/v2/user/me)";
        HttpHeaders headers = new HttpHeaders();
        headers.add("Authorization", "Bearer " + accessToken);

        RestTemplate template = new RestTemplate();
        ResponseEntity<KaKaoUserDto> response = template.exchange(
                requestUri, HttpMethod.POST, new HttpEntity<>(headers), KaKaoUserDto.class);

        KaKaoUserDto responseBody = response.getBody();
        log.info("카카오 사용자 정보 조회 성공: {}", responseBody);
        
        // TODO: 이 정보를 바탕으로 DB에서 사용자를 찾거나 새로 생성하고, 자체 JWT를 발급하여 쿠키로 전달해야 함.
    }
}
```

### 4.5. DTO (`KaKaoUserDto.java`)

카카오 사용자 정보 API의 응답(JSON)을 매핑하기 위한 DTO(Data Transfer Object)입니다. `@JsonProperty`를 사용하여 JSON 필드명과 자바 필드명을 매핑합니다.

```java
// backend/src/main/java/com/study/mate/service/KaKaoUserDto.java
package com.study.mate.service;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.*;
import java.time.LocalDateTime;

@Getter
@ToString
@NoArgsConstructor
@AllArgsConstructor
public class KaKaoUserDto {
    private Long id;

    @JsonProperty("connected_at")
    private LocalDateTime connectedAt;

    private Properties properties;

    @Getter @ToString
    public static class Properties {
        private String nickname;
        @JsonProperty("profile_image")
        private String profileImage;
    }
}
```

-----

## 5\. 데이터베이스 엔티티

소셜 로그인을 지원하기 위해 사용자 엔티티에 `provider`와 `providerId` 필드를 추가합니다.

### `User.java`

```java
// backend/src/main/java/com/study/mate/entity/User.java
package com.study.mate.entity;

import jakarta.persistence.*;
import lombok.*;

@Getter
@Builder
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor
@Entity
@Table(
    name = "users",
    uniqueConstraints = {
        @UniqueConstraint(name = "uk_user_email", columnNames = {"email"}),
        // provider와 providerId 조합이 유니크하도록 제약조건 추가
        @UniqueConstraint(name = "uk_provider_provider_id", columnNames = {"provider", "provider_id"})
    }
)
public class User extends BaseTimeEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;
    private String nickname;
    private String profileImageUrl;

    @Enumerated(EnumType.STRING) // "GOOGLE", "KAKAO" 등 문자열로 저장
    private Provider provider;

    private String providerId; // 소셜 로그인 제공업체의 사용자 고유 ID
    // ...
}
```

### `Provider.java`

로그인 제공자를 구분하기 위한 `Enum` 타입입니다.

```java
// backend/src/main/java/com/study/mate/entity/Provider.java
package com.study.mate.entity;

public enum Provider {
    GOOGLE,
    KAKAO
}
```

```
```