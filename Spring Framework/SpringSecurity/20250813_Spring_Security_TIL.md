
# Spring Security & JWT 기반 Full-Stack (심화)

## 1\. 개요: 인증/인가 전체 흐름 

이 프로젝트의 인증 및 인가 흐름은 JWT(JSON Web Token)를 기반으로 동작합니다.

1.  **최초 인증 (로그인)**

    1.  **클라이언트 → 서버**: 사용자가 아이디/비밀번호로 로그인 요청 (`/api/auth/login`)
    2.  **`AuthController`**: 요청 수신 후 `UserService`에 전달
    3.  **`UserService`**:
          * `UserRepository`를 통해 사용자 정보 조회
          * `PasswordEncoder`로 비밀번호 일치 여부 검증
          * 인증 성공 시, `JwtProvider`를 통해 **JWT 토큰** 생성
    4.  **서버 → 클라이언트**: 생성된 JWT 토큰을 `AuthResponse` DTO에 담아 응답

2.  **후속 요청 (인가)**

    1.  **클라이언트 → 서버**: 발급받은 JWT 토큰을 HTTP 요청 헤더(`Authorization: Bearer <token>`)에 담아 API 요청
    2.  **`JwtAuthenticationFilter`**: 서버에 도달하는 모든 요청을 가로채 토큰의 유효성을 검증
    3.  **토큰 검증 성공 시**: `SecurityContextHolder`에 사용자 인증 정보를 등록. 이를 통해 Spring Security는 해당 요청이 **인증된 사용자**의 요청임을 인지합니다.
    4.  **`Controller`**: `@AuthenticationPrincipal`을 통해 인증된 사용자 정보에 접근하여 비즈니스 로직 처리

-----

## 2\. 인증 및 인가 (Authentication & Authorization) 

JWT를 사용하여 서버가 상태를 갖지 않는(Stateless) 인증/인가 시스템을 구축합니다.

### 2.1. JWT 구현 (`jwt` 패키지)

#### `JwtProvider.java` - 토큰 생성 및 검증기

JWT 토큰의 생성, 서명, 유효성 검증, 정보 추출을 담당하는 핵심 클래스입니다.

```java
package com.spring.toyproject.jwt;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtException;
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
     * @param username - 발급 대상의 사용자 이름 (유일하게 사용자를 식별할 값)
     * @return - 암호화된 JWT 토큰 문자열
     */
    public String generateToken(String username) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtProperties.getExpiration());

        return Jwts.builder()
                .subject(username) // sub: 토큰의 주체 (누구의 토큰인가)
                .issuedAt(now) // iat: 토큰 발급 시간
                .expiration(expiryDate) // exp: 토큰 만료 시간
                .issuer("Toy Project By ") // iss: 토큰 발급자
                .signWith(getSigningKey()) // 서명
                .compact();
    }

    /**
     * JWT 토큰의 유효성을 검증합니다.
     * 서명, 만료 시간 등을 확인합니다.
     * @param token 검증할 토큰 문자열
     * @return 유효하면 true, 아니면 false
     */
    public boolean validateToken(String token) {
        try {
            // 토큰을 파싱하여 Claims를 추출. 이 과정에서 서명 검증이 자동으로 이루어집니다.
            // 만료 시간이 지났거나, 서명이 잘못된 경우 JwtException 발생.
            getClaimsFromToken(token);
            return true;
        } catch (JwtException e) {
            log.warn("Invalid JWT token: {}", e.getMessage());
            return false;
        }
    }

    /**
     * 토큰에서 사용자 이름(subject)을 추출합니다.
     * @param token 정보룰 추출할 토큰
     * @return 사용자 이름 (username)
     */
    public String getUsernameFromToken(String token) {
        return getClaimsFromToken(token).getSubject();
    }

    /**
     * JWT 토큰을 파싱하여 실제 데이터(Claims)를 추출합니다.
     */
    private Claims getClaimsFromToken(String token) {
        return Jwts.parser()
                .verifyWith(getSigningKey()) // 1. 서명 검증: 이 서버가 발급한 토큰이 맞는지 확인
                .build()
                .parseSignedClaims(token)    // 2. 파싱: 토큰 문자열을 Claims 객체로 변환 (위변조 시 예외 발생)
                .getPayload();
    }

    /**
     * JWT 토큰 발급에 필요한 서명 키(SecretKey)를 생성합니다.
     * 이 키는 외부에 노출되어서는 안 됩니다.
     */
    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(jwtProperties.getSecret().getBytes(StandardCharsets.UTF_8));
    }
}
```

#### `JwtAuthenticationFilter.java` - 인증 필터

클라이언트의 모든 요청에 대해 가장 먼저 실행되어 JWT 토큰을 검사하는 필터입니다. `OncePerRequestFilter`를 상속하여 한 요청에 대해 단 한 번만 실행됨을 보장합니다.

```java
package com.spring.toyproject.jwt;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.ArrayList;

/**
 * JWT 인증 필터
 * 클라이언트의 모든 요청에 대해 토큰을 검사하고 인증 정보를 설정하는 자동화된 필터
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtProvider jwtProvider;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        try {
            // 1. 요청 헤더에서 토큰 추출
            String token = extractTokenFromHeader(request);

            // 2. 토큰 유효성 검사
            if (token != null && jwtProvider.validateToken(token)) {
                // 3. 토큰에서 사용자명 추출
                String username = jwtProvider.getUsernameFromToken(token);

                // 4. Spring Security를 위한 인증 객체 생성
                // 추가 주석: 여기서는 권한(Authority)을 설정하지 않았지만, 필요 시 토큰에 권한 정보를 담고 여기서 설정할 수 있습니다.
                UsernamePasswordAuthenticationToken auth = new UsernamePasswordAuthenticationToken(
                        username, // Principal: 컨트롤러에서 @AuthenticationPrincipal로 사용할 사용자 식별자
                        null,     // Credentials(비밀번호): 토큰 인증 방식에서는 보통 null로 설정
                        new ArrayList<>() // Authorities(권한 목록)
                );

                // 5. Spring Security의 컨텍스트 홀더에 인증 정보 등록
                // 추가 주석: 이 작업이 완료되어야 Spring Security가 해당 사용자를 '인증된 사용자'로 간주합니다.
                SecurityContextHolder.getContext().setAuthentication(auth);
                log.debug("JWT 인증 성공: {}", username);
            }
        } catch (Exception e) {
            log.error("JWT 인증 오류 발생: {}", e.getMessage());
        }

        // 6. 다음 필터로 요청 전달
        filterChain.doFilter(request, response);
    }

    /**
     * 요청 헤더(Authorization)에서 Bearer 토큰을 추출합니다.
     */
    private String extractTokenFromHeader(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        // 추가 주석: StringUtils.hasText는 null, 빈 문자열, 공백 문자열을 모두 체크해주는 유용한 유틸리티입니다.
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7); // "Bearer " 다음의 토큰 문자열만 반환
        }
        return null;
    }
}
```

### 2.2. Spring Security 설정 (`config` 패키지)

#### `SecurityConfig.java` - 보안 규칙 정의 (심화)

애플리케이션의 보안 규칙을 최종적으로 설정합니다. 어떤 요청을 허용하고, 어떤 요청에 인증이 필요한지 정의하며, 우리가 만든 `JwtAuthenticationFilter`를 등록합니다.

```java
package com.spring.toyproject.config;

import com.spring.toyproject.jwt.JwtAuthenticationFilter;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

/**
 * Spring Security 설정 클래스
 * 인증 설정 및 기본 보안 규칙 설정
 */
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {

        http
                // CSRF 공격 방지 기능 비활성화 (Stateless 서버에서는 불필요)
                .csrf(AbstractHttpConfigurer::disable)
                // CORS 설정은 기본값을 따름 (필요시 별도 설정 가능)
                .cors(cors -> cors.configure(http))
                // 세션 관리 정책을 STATELESS로 설정 (서버가 세션을 사용하지 않음)
                .sessionManagement(session ->
                        session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                // 기본 제공 로그인 폼 비활성화
                .formLogin(AbstractHttpConfigurer::disable)
                // HTTP Basic 인증 비활성화
                .httpBasic(AbstractHttpConfigurer::disable)


                // 경로별 인가(Authorization) 규칙 설정
                .authorizeHttpRequests(
                        auth -> auth
                                // 아래 경로들은 인증 없이 누구나 접근 허용 (permitAll)
                                .requestMatchers(
                                        "/", "/login", "/signup", "/dashboard",
                                        "/trips/**", "/css/**", "/js/**", "/images/**", "/favicon.ico"
                                ).permitAll()
                                .requestMatchers("/api/auth/**").permitAll() // 인증 관련 API는 모두 허용

                                // '/api/**' 패턴의 경로는 반드시 '인증된(authenticated)' 사용자만 접근 가능
                                .requestMatchers("/api/**").authenticated()

                                // 위에서 지정한 경로 외의 모든 요청은 인증이 필요함
                                .anyRequest().authenticated()
                )


                // 커스텀 필터인 JwtAuthenticationFilter를 UsernamePasswordAuthenticationFilter 앞에 추가
                // 추가 주석: 모든 요청이 Spring Security의 기본 인증(id/pw) 필터에 도달하기 전에
                // JWT 토큰을 먼저 검사하도록 설정하는 것이 핵심입니다.
                .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

#### `PasswordEncoderConfig.java` - 비밀번호 암호화

`BCryptPasswordEncoder`를 사용하여 비밀번호를 안전하게 해싱합니다. (이전 내용과 동일)

-----

## 3\. 동적 쿼리와 QueryDSL 

정적인 쿼리가 아닌, 사용자의 입력(검색 조건, 정렬 기준)에 따라 동적으로 SQL을 생성해야 할 때 QueryDSL을 사용하면 타입-세이프(Type-Safe)하고 가독성 높은 코드를 작성할 수 있습니다.

### 3.1. 설정 및 기본 구조

#### `QueryDslConfig.java` - QueryDSL 설정

QueryDSL 쿼리를 생성하고 실행하는 핵심 객체인 `JPAQueryFactory`를 Spring 컨테이너에 Bean으로 등록합니다.

```java
package com.spring.toyproject.config;

import com.querydsl.jpa.impl.JPAQueryFactory;
import jakarta.persistence.EntityManager;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * QueryDSL 설정 클래스
 * JPAQueryFactory를 Spring Bean으로 등록하여 QueryDSL 사용을 위한 설정
 */
@Configuration
@RequiredArgsConstructor
public class QueryDslConfig {

    private final EntityManager entityManager;

    /**
     * JPAQueryFactory Bean을 등록하여 다른 Repository에서 주입받아 사용할 수 있게 합니다.
     */
    @Bean
    public JPAQueryFactory jpaQueryFactory() {
        return new JPAQueryFactory(entityManager);
    }
}
```

#### `TripRepositoryCustom.java` & `TripRepository.java`

1.  **`TripRepositoryCustom`**: QueryDSL을 사용할 커스텀 메서드를 정의하는 인터페이스입니다.
2.  **`TripRepository`**: Spring Data JPA의 `JpaRepository`와 우리가 만든 `TripRepositoryCustom`을 함께 상속받습니다.

<!-- end list -->

```java
// TripRepositoryCustom.java
package com.spring.toyproject.repository.custom;

import com.spring.toyproject.domain.entity.Trip;
import com.spring.toyproject.domain.entity.User;
// ... (imports) ...
public interface TripRepositoryCustom {
    // 동적 쿼리로 검색 조건별 여행 목록 조회 (페이징 포함)
    Page<Trip> getTripList(User user, TripSearchCondition condition, Pageable pageable);

    @Getter
    @Builder
    class TripSearchCondition { /* 검색 조건 필드들 */ }
}

// TripRepository.java
package com.spring.toyproject.repository.base;

// ... (imports) ...
public interface TripRepository extends JpaRepository<Trip, Long>, TripRepositoryCustom {
    // JpaRepository의 기본 CRUD 메서드와 TripRepositoryCustom의 커스텀 메서드를 모두 사용 가능
}
```

### 3.2. QueryDSL 구현체

#### `TripRepositoryImpl.java` - 동적 쿼리 구현

`TripRepositoryCustom` 인터페이스를 구현하여 실제 QueryDSL 코드를 작성합니다.

  - **`BooleanBuilder`**: `if` 문을 사용하여 조건이 있을 때만 WHERE 절에 `AND` 조건을 동적으로 추가합니다.
  - **`OrderSpecifier`**: `switch` 문을 사용하여 정렬 기준에 따라 동적으로 ORDER BY 절을 생성합니다.

<!-- end list -->

```java
package com.spring.toyproject.repository.impl;

import com.querydsl.core.BooleanBuilder;
import com.querydsl.core.types.OrderSpecifier;
import com.querydsl.jpa.impl.JPAQueryFactory;
import com.spring.toyproject.domain.entity.Trip;
import com.spring.toyproject.domain.entity.User;
import com.spring.toyproject.repository.custom.TripRepositoryCustom;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageImpl;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Repository;

import java.util.List;
import static com.spring.toyproject.domain.entity.QTrip.trip; // Q-Type 클래스 import

/**
 * TripRepositoryCustom의 구현체
 * QueryDSL을 사용하여 동적 쿼리 로직을 작성합니다.
 */
@Repository
@RequiredArgsConstructor
public class TripRepositoryImpl implements TripRepositoryCustom {

    private final JPAQueryFactory factory;

    @Override
    public Page<Trip> getTripList(User user, TripSearchCondition condition, Pageable pageable) {

        // WHERE 절을 동적으로 구성하기 위한 BooleanBuilder
        BooleanBuilder whereClause = new BooleanBuilder();
        whereClause.and(trip.user.eq(user)); // 기본 조건: 현재 로그인한 사용자의 여행만

        // 1. 상태(status) 검색 조건 추가
        if (condition.getStatus() != null) {
            whereClause.and(trip.status.eq(condition.getStatus()));
        }
        // 2. 목적지(destination) 검색 조건 추가 (대소문자 무시)
        if (condition.getDestination() != null && !condition.getDestination().trim().isEmpty()) {
            whereClause.and(trip.destination.containsIgnoreCase(condition.getDestination()));
        }
        // 3. 제목(title) 검색 조건 추가 (대소문자 무시)
        if (condition.getTitle() != null && !condition.getTitle().trim().isEmpty()) {
            whereClause.and(trip.title.containsIgnoreCase(condition.getTitle()));
        }

        // 데이터 목록 조회 쿼리
        List<Trip> tripList = factory
                .selectFrom(trip)
                .where(whereClause)
                .orderBy(getOrderSpecifier(condition)) // 동적 정렬
                .offset(pageable.getOffset()) // 페이징 시작 위치
                .limit(pageable.getPageSize())  // 한 페이지에 가져올 개수
                .fetch();

        // 전체 카운트 조회 쿼리 (페이징을 위해 필요)
        Long totalCount = factory
                .select(trip.count())
                .from(trip)
                .where(whereClause)
                .fetchOne();

        // Page 객체로 변환하여 반환
        return new PageImpl<>(tripList, pageable, totalCount == null ? 0L : totalCount);
    }

    // 정렬 조건을 동적으로 생성하는 헬퍼 메서드
    private OrderSpecifier<?> getOrderSpecifier(TripSearchCondition condition) {
        String sortBy = condition.getSortBy();
        String sortDirection = condition.getSortDirection();

        switch (sortBy.toLowerCase()) {
            case "startdate":
                return sortDirection.equalsIgnoreCase("DESC") ? trip.startDate.desc() : trip.startDate.asc();
            case "enddate":
                return sortDirection.equalsIgnoreCase("DESC") ? trip.endDate.desc() : trip.endDate.asc();
            case "title":
                return sortDirection.equalsIgnoreCase("DESC") ? trip.title.desc() : trip.title.asc();
            case "destination":
                return sortDirection.equalsIgnoreCase("DESC") ? trip.destination.desc() : trip.destination.asc();
            default: // 기본 정렬: 최신순
                return sortDirection.equalsIgnoreCase("DESC") ? trip.createdAt.desc() : trip.createdAt.asc();
        }
    }
}
```

-----

## 4\. API 계층 (Controller) 

### `TripController.java` - 여행 관련 API

인증된 사용자를 위한 여행(Trip) 관련 CRUD API를 제공합니다.

| Annotation | 설명 |
| :--- | :--- |
| `@AuthenticationPrincipal` | Spring Security 컨텍스트에 저장된 현재 인증된 사용자의 Principal(여기서는 `username`)을 매개변수로 직접 주입받습니다. |

```java
package com.spring.toyproject.api;

// ... (imports) ...

@RestController
@RequiredArgsConstructor
@Slf4j
@RequestMapping("/api/trips")
public class TripController {

    private final TripService tripService;

    /**
     * 여행 생성 API
     * POST /api/trips
     */
    @PostMapping
    public ResponseEntity<?> createTrip(
            @RequestBody @Valid TripRequest request,
            // 추가 주석: JwtAuthenticationFilter에서 SecurityContextHolder에 저장한 사용자명(username)이 여기에 주입됩니다.
            @AuthenticationPrincipal String username
    ) {
        log.info("여행 생성 API 호출 - 사용자: {}", username);
        Trip response = tripService.createTrip(request, username);
        return ResponseEntity.ok()
                .body(ApiResponse.success("여행이 성공적으로 생성되었습니다.", response));
    }

    /**
     * 사용자별 여행 목록 조회 API (동적 쿼리)
     * GET /api/trips?page=0&size=5&sortBy=startDate&status=PLANNING
     */
    @GetMapping
    public ResponseEntity<ApiResponse<Page<TripListItemDto>>> getUserTrips(
            @AuthenticationPrincipal String username,
            // 추가 주석: 쿼리 파라미터를 DTO 객체로 한번에 바인딩하여 편리하게 사용합니다.
            TripSearchRequestDto request) {

        log.info("사용자별 여행 목록 조회 API 호출 - 사용자: {}, 페이지: {}, 크기: {}",
                username, request.getPage(), request.getSize());

        TripRepositoryCustom.TripSearchCondition condition = request.toCondition();
        Pageable pageable = PageRequest.of(request.getPage(), request.getSize());
        Page<TripListItemDto> trips = tripService.getUserTripsList(username, condition, pageable);

        return ResponseEntity.ok(ApiResponse.success("여행 정보 목록이 조회되었습니다.", trips));
    }
}
```

-----

## 5\. 서비스 계층 (Service) 

### `TripService.java` - 여행 관련 비즈니스 로직

여행 정보 생성 및 조회의 핵심 비즈니스 로직을 처리합니다.

```java
package com.spring.toyproject.service;

// ... (imports) ...

@Service
@Slf4j
@RequiredArgsConstructor
@Transactional
public class TripService {

    private final TripRepository tripRepository;
    private final UserRepository userRepository;

    /**
     * 여행 정보 생성
     */
    public Trip createTrip(TripRequest request, String username) {
        log.info("여행 생성 시작! - 사용자명: {}, 제목: {}", username, request.getTitle());

        // 토큰에서 파싱한 사용자명으로 User 엔티티 조회
        User foundUser = userRepository.findByUsername(username)
                .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND));

        // DTO를 Entity로 변환하여 저장
        Trip trip = Trip.builder()
                .title(request.getTitle())
                // ... (다른 필드들)
                .user(foundUser) // JPA에서는 연관관계의 주인에 엔티티 객체를 통째로 넣어줘야 합니다.
                .build();

        Trip savedTrip = tripRepository.save(trip);
        log.info("여행 생성 완료 - 여행 ID: {}", savedTrip.getId());
        return savedTrip;
    }

    /**
     * 사용자별 여행 목록 조회 (DTO로 변환하여 반환)
     */
    @Transactional(readOnly = true) // 단순 조회 작업이므로 readOnly=true 옵션으로 성능 최적화
    public Page<TripListItemDto> getUserTripsList(String username, TripRepositoryCustom.TripSearchCondition condition, Pageable pageable) {
        log.info("사용자별 여행 목록 조회 - 사용자명: {}", username);

        User user = userRepository.findByUsername(username)
                .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND));

        // Repository의 동적 쿼리 메서드 호출
        Page<Trip> tripPage = tripRepository.getTripList(user, condition, pageable);

        // Page<Trip>을 Page<TripListItemDto>로 변환하여 반환
        return tripPage.map(TripListItemDto::from);
    }
}
```

-----

이 외에 `PageController`는 단순 페이지 렌더링, 다양한 DTO 클래스들은 각 계층 간의 데이터 전달 역할을 충실히 수행하고 있습니다. 이번 학습을 통해 JWT 기반 인증/인가 흐름과 QueryDSL을 이용한 동적 쿼리 작성법에 대해 깊이 이해하셨을 것 같습니다. 수고하셨습니다\!**
