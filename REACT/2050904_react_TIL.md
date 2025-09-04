# Event Fullstack Application 개발 학습 노트

## 1\. 회원가입 기능 구현 (FE & BE)

### 1.1. 프론트엔드: 회원가입 UI 및 로직

#### 1.1.1. `PasswordInput.jsx`: 비밀번호 입력 및 유효성 검사

  - **역할**: 회원가입 절차 중 비밀번호를 입력받고, 실시간으로 유효성을 검사하는 컴포넌트입니다.
  - **주요 기능**:
      - `useState`를 사용하여 비밀번호 값(`password`)과 에러 메시지(`errorMessage`) 상태를 관리합니다.
      - `useRef`를 사용하여 입력창에 자동으로 포커스를 줍니다.
      - 비밀번호는 **8자 이상, 숫자, 문자, 특수문자를 모두 포함**해야 유효한 것으로 간주합니다.
      - `onChange` 이벤트 핸들러(`changeHandler`) 내에서 유효성 검사를 수행하고, 결과에 따라 부모 컴포넌트(`SignUpForm`)로 `onSuccess` 콜백 함수를 호출하여 상태(유효 여부, 입력된 비밀번호)를 전달합니다.
      - 유효하지 않을 경우, 에러 메시지를 화면에 표시합니다.

<!-- end list -->

```jsx
// frontend/src/components/auth/PasswordInput.jsx

import React, { useEffect, useRef, useState } from 'react';
import styles from './SignUpForm.module.scss';

const PasswordInput = ({ onSuccess }) => {
    const passwordRef = useRef();

    const [password, setPassword] = useState('');
    const [errorMessage, setErrorMessage] = useState('');

    // 패스워드 패턴검증
    const validatePassword = (password) => {
        const passwordPattern =
            /^(?=.*[A-Za-z])(?=.*\d)(?=.*[@$!%*#?&])[A-Za-z\d@$!%*#?&]{8,}$/;
        return passwordPattern.test(password);
    };

    const changeHandler = (e) => {
        const newPassword = e.target.value;
        setPassword(newPassword);

        if (validatePassword(newPassword)) {
            setErrorMessage('');
            // 유효성 검사 결과와 비밀번호를 부모 컴포넌트로 전달
            onSuccess(true, newPassword);
        } else {
            setErrorMessage(
                '비밀번호는 8자 이상이며, 숫자, 문자, 특수문자를 모두 포함해야 합니다.'
            );
            onSuccess(false, newPassword);
        }
    };

    useEffect(() => {
        passwordRef.current.focus();
    }, []);

    return (
        <>
            <p>Step 3: 안전한 비밀번호를 설정해주세요.</p>
            <input
                ref={passwordRef}
                type='password'
                value={password}
                onChange={changeHandler}
                className={errorMessage ? styles.invalidInput : ''}
                placeholder='Enter your password'
            />
            {errorMessage && <p className={styles.errorMessage}>{errorMessage}</p>}
        </>
    );
};

export default PasswordInput;
```

#### 1.1.2. `SignUpForm.jsx`: 회원가입 절차 관리

  - **역할**: 이메일 입력, 인증번호 확인, 비밀번호 설정 등 회원가입의 전체 단계를 관리하는 폼 컴포넌트입니다.
  - **주요 기능**:
      - `useState`를 통해 현재 진행 단계(`step`), 다음 단계 진행 상태(`isNext`), 회원가입 버튼 활성화 여부(`isActiveButton`) 등을 관리합니다.
      - `PasswordInput` 컴포넌트로부터 유효성 검사 결과를 받아(`passwordSuccessHandler`), 회원가입 완료 버튼을 활성화합니다.
      - '회원가입 완료' 버튼 클릭 시 `handleSubmit` 함수가 실행되어, 백엔드 서버에 최종 회원가입 요청을 보냅니다.

<!-- end list -->

```jsx
// frontend/src/components/auth/SignUpForm.jsx (일부)

// ... (import 및 다른 코드)

const SignUpForm = () => {
    // ... (다른 상태 관리)
    const [enteredPassword, setEnteredPassword] = useState('');
    const [isActiveButton, setIsActiveButton] = useState(false); // 회원가입 버튼 활성화 여부
    const [step, setStep] = useState(1);

    // ... (다른 핸들러)

    // 패스워드 입력이 끝날 때 호출될 함수
    const passwordSuccessHandler = (isValid, password) => {
        setEnteredPassword(password);
        // 회원가입버튼을 열어줄지 여부
        setIsActiveButton(isValid);
    };

    // 회원가입 완료 이벤트
    const handleSubmit = e => {
        e.preventDefault();

        (async () => {
            const response = await fetch(`${AUTH_API_URL}/join`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    email: enteredEmail,
                    password: enteredPassword
                })
            });

            if (!response.ok) throw new Error('회원가입에 실패했습니다.');

            const data = await response.json();
            alert(data.message);
            navigate('/');
        })();
    };

    return (
        <div className={styles.signupForm}>
            <div className={styles.formStepActive}>
                {step === 1 && <EmailInput /* ... */ />}
                {step === 2 && <VerificationInput /* ... */ />}
                {step === 3 && <PasswordInput onSuccess={passwordSuccessHandler} />}

                {isActiveButton && (
                    <div>
                        <button onClick={handleSubmit}>회원가입 완료</button>
                    </div>
                )}
                {/* ... */}
            </div>
        </div>
    );
};

export default SignUpForm;
```

-----

### 1.2. 백엔드: 회원가입 처리 및 보안

#### 1.2.1. `PasswordEncoderConfig.java`: 비밀번호 암호화 설정

  - **역할**: 사용자의 비밀번호를 안전하게 저장하기 위해 암호화하는 `PasswordEncoder`를 Spring 컨테이너에 빈으로 등록합니다.
  - **`BCryptPasswordEncoder`**: 강력한 해싱 알고리즘 중 하나인 **BCrypt**를 사용하는 구현체입니다. 비밀번호를 단방향으로 암호화하며, 원본 값을 유추하기 어렵게 만듭니다.

<!-- end list -->

```java
// backend/src/main/java/com/study/event/config/PasswordEncoderConfig.java

package com.study.event.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class PasswordEncoderConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

#### 1.2.2. `EventUserService.java`: 회원가입 마무리 로직

  - **역할**: 프론트엔드로부터 받은 회원 정보를 최종적으로 데이터베이스에 저장하는 서비스 로직을 담당합니다.
  - **`confirmSignup` 메서드**:
    1.  `SignupRequest` DTO로 이메일과 비밀번호를 전달받습니다.
    2.  이메일을 사용해 데이터베이스에서 **임시 회원가입 상태**의 사용자 정보를 조회합니다.
    3.  이메일 인증이 완료되었는지 확인합니다.
    4.  `PasswordEncoder`를 사용해 전달받은 비밀번호를 **암호화**합니다.
    5.  암호화된 비밀번호와 생성 시간을 사용자 정보에 업데이트하고, 데이터베이스에 저장하여 회원가입을 완료합니다.

<!-- end list -->

```java
// backend/src/main/java/com/study/event/service/EventUserService.java (일부)

// ... (import 및 다른 코드)

@Service
@Slf4j
@RequiredArgsConstructor
@Transactional
public class EventUserService {

    private final PasswordEncoder passwordEncoder;
    private final EventUserRepository eventUserRepository;
    // ... (다른 의존성)

    // 회원가입 마무리 처리
    public void confirmSignup(SignupRequest dto) {

        // 임시 회원가입된 회원정보를 조회
        EventUser foundUser = eventUserRepository.findByEmail(dto.email()).orElseThrow(
                () -> new RuntimeException("임시 회원가입된 정보가 없습니다.")
        );
        // 이메일인증을 받았는지 확인
        if (!foundUser.isEmailVerified()) {
            throw new RuntimeException("이메일 인증이 완료되지 않은 회원입니다.");
        }

        // 데이터베이스에 임시 회원가입된 회원정보의 패스워드와 생성시간을 채워넣기
        foundUser.confirm(passwordEncoder.encode(dto.password()));
        eventUserRepository.save(foundUser);
    }
    // ... (다른 메서드)
}
```

#### 1.2.3. `AuthController.java`: 회원가입 API 엔드포인트

  - **역할**: 회원가입 요청을 받는 REST API 컨트롤러입니다.
  - **`/join` 엔드포인트**:
      - `@PostMapping("/join")` 어노테이션을 통해 POST 요청을 처리합니다.
      - `@RequestBody`로 클라이언트가 보낸 JSON 데이터를 `SignupRequest` DTO 객체로 변환합니다.
      - `EventUserService`의 `confirmSignup` 메서드를 호출하여 회원가입 로직을 수행합니다.
      - 성공 시, "회원가입이 완료되었습니다." 메시지를 담아 `200 OK` 응답을 반환합니다.

<!-- end list -->

```java
// backend/src/main/java/com/study/event/api/AuthController.java (일부)

// ... (import 및 다른 코드)

@RestController
@Slf4j
@RequiredArgsConstructor
@RequestMapping("/api/auth")
public class AuthController {

    private final EventUserService eventUserService;

    // ... (다른 API)

    // 회원가입 마무리 요청
    @PostMapping("/join")
    public ResponseEntity<?> join(@RequestBody SignupRequest dto) {

        log.info("save request user info - {}", dto);
        eventUserService.confirmSignup(dto);

        return ResponseEntity.ok().body(Map.of(
                "message", "회원가입이 완료되었습니다."
        ));
    }
    // ...
}
```

-----

## 2\. 로그인 및 인증 (JWT)

### 2.1. 백엔드: 로그인 검증 및 토큰 발급

#### 2.1.1. `EventUserService.java`: 로그인 인증 로직

  - **`authenticate` 메서드**:
    1.  `LoginRequest` DTO로 전달받은 이메일로 회원을 조회합니다.
    2.  회원이 존재하지 않거나, 이메일 인증이 완료되지 않은 경우 예외를 발생시킵니다.
    3.  `passwordEncoder.matches()` 메서드를 사용하여, 입력된 평문 비밀번호와 데이터베이스에 저장된 **암호화된 비밀번호**를 비교합니다.
    4.  인증에 성공하면 `JwtTokenProvider`를 통해 \*\*액세스 토큰(Access Token)\*\*을 생성합니다.
    5.  생성된 토큰, 이메일, 역할(Role) 정보를 담아 클라이언트에 반환합니다.

<!-- end list -->

```java
// backend/src/main/java/com/study/event/service/EventUserService.java (일부)

// ...
public class EventUserService {

    private final PasswordEncoder passwordEncoder;
    private final JwtTokenProvider tokenProvider;
    private final EventUserRepository eventUserRepository;
    // ...

    // 로그인 검증하기
    public Map<String, Object> authenticate(LoginRequest dto) {

        // 이메일을 통해 회원가입 여부 확인
        EventUser foundUser
                = eventUserRepository.findByEmail(dto.email()).orElseThrow(
                () -> new RuntimeException("가입된 회원이 아닙니다.")
        );

        // 회원가입을 중단한 회원에 대해서
        if (!foundUser.isEmailVerified() || foundUser.getPassword() == null) {
            throw new RuntimeException("회원가입이 완료되지 않은 회원입니다. 다시 가입해주세요.");
        }
        // 패스워드 일치 검사
        if (!passwordEncoder.matches(dto.password(), foundUser.getPassword())) {
            throw new RuntimeException("비밀번호가 틀렸습니다.");
        }

        // 로그인 성공 - 토큰 발급
        String accessToken = tokenProvider.createAccessToken(dto.email());

        return Map.of(
                "token", accessToken,
                "message", "로그인에 성공했습니다.",
                "email", dto.email(),
                "role", foundUser.getRole().toString()
        );
    }
    // ...
}
```

#### 2.1.2. `JwtTokenProvider.java`: JWT 토큰 생성 및 검증

  - **역할**: JWT(JSON Web Token)를 생성, 파싱, 검증하는 핵심 로직을 담당합니다.
  - **주요 메서드**:
      - `createAccessToken()`: 사용자 이메일을 받아 액세스 토큰을 생성합니다.
      - `validateToken()`: 토큰의 유효성(서명, 만료 시간 등)을 검증합니다.
      - `getCurrentLoginUserEmail()`: 유효한 토큰에서 사용자 이메일 정보를 추출합니다.
  - **토큰 생성 과정**:
    1.  `application.yml`에 설정된 `secret-key`와 유효 기간(`validity-time`)을 `JwtProperties`를 통해 읽어옵니다.
    2.  `Jwts.builder()`를 사용하여 토큰의 내용(발급자, 만료 시간, 주제(이메일))을 설정합니다.
    3.  `signWith()` 메서드로 비밀키를 사용해 토큰에 서명하여 위변조를 방지합니다.
    4.  `compact()`로 최종적인 문자열 형태의 JWT를 생성합니다.

<!-- end list -->

```java
// backend/src/main/java/com/study/event/jwt/JwtTokenProvider.java

package com.study.event.jwt;

// ... (imports)

@Component
@RequiredArgsConstructor
@Slf4j
public class JwtTokenProvider {

    private final JwtProperties jwtProperties;
    private SecretKey key; // 비밀키

    @PostConstruct
    public void init() {
        // Base64로 인코딩된 key를 디코딩 후, HMAC-SHA 알고리즘으로 다시 암호화
        this.key = Keys.hmacShaKeyFor(Decoders.BASE64.decode(jwtProperties.getSecretKey()));
    }

    // 액세스 토큰 생성
    public String createAccessToken(String email) {
        return createToken(email, jwtProperties.getAccessTokenValidityTime());
    }

    // 공통 토큰 생성 로직
    private String createToken(String email, long validityTime) {
        Date now = new Date();
        Date validity = new Date(now.getTime() + validityTime);

        return Jwts.builder()
                .setIssuer("Event API")      // 발급자 정보
                .setIssuedAt(now)            // 발급시간
                .setExpiration(validity)     // 만료시간
                .setSubject(email)           // 토큰을 구별할 유일한 값 (이메일)
                .signWith(key)               // 서명 포함
                .compact();
    }

    // 토큰 유효성 검증
    public boolean validateToken(String token) {
        try {
            parseClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            log.warn("invalid token: {}", e.getMessage(), e);
            return false;
        }
    }

    // 토큰에서 이메일 추출
    public String getCurrentLoginUserEmail(String token) {
        return parseClaims(token).getSubject();
    }

    private Claims parseClaims(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(key)
                .build()
                .parseClaimsJws(token)
                .getBody();
    }
}
```

-----

### 2.2. 프론트엔드: 로그인 요청 및 상태 관리

#### 2.2.1. `events-actions.js`: React Router `action` 함수

  - **`loginAction`**:
      - `LoginForm`에서 `POST` 요청이 발생했을 때 트리거되는 **액션 함수**입니다.
      - `request.formData()`를 통해 폼 데이터를 읽어와 `payload`를 구성합니다.
      - `fetch`를 사용해 백엔드 로그인 API(`api/auth/login`)를 호출합니다.
      - **로그인 실패 시 (422 Unprocessable Entity)**: 백엔드가 보낸 에러 메시지를 `return`합니다. 이 값은 `LoginForm` 컴포넌트에서 `useActionData` 훅으로 받아 화면에 표시할 수 있습니다.
      - **로그인 성공 시**: 서버로부터 받은 사용자 데이터(토큰 포함)를 `localStorage`에 `userData`라는 키로 저장합니다.
      - `redirect('/')`를 반환하여 사용자를 홈 페이지로 이동시킵니다.

<!-- end list -->

```javascript
// frontend/src/loader/events-actions.js (일부)

import { redirect } from 'react-router-dom';
import { AUTH_API_URL } from '../config/host-config.js';

// 로그인 처리 액션함수
export const loginAction = async ({ request }) => {

    // 입력데이터 읽기
    const formData = await request.formData();
    const payload = {
        email: formData.get('email'),
        password: formData.get('password'),
    };

    const response = await fetch(`${AUTH_API_URL}/login`, {
        method: 'POST',
        headers: { 'Content-Type':'application/json' },
        body: JSON.stringify(payload)
    });

    const data = await response.json();

    if (response.status === 422) {
        // 서버에서 응답한 데이터를 컴포넌트에서 가져다 사용할 수 있게 데이터를 리턴.
        // action함수를 처리하는 컴포넌트는 useActionData라는 훅으로 사용가능
        return data.message;
    }

    // 로그인에 성공했을 때 - 토큰을 저장
    localStorage.setItem('userData', JSON.stringify(data));

    return redirect('/');
};

// ... (다른 action 함수)
```

#### 2.2.2. `LoginForm.jsx`: 로그인 폼 UI

  - React Router의 `<Form>` 컴포넌트를 사용하여 `POST` 요청을 생성합니다. 이 폼이 제출되면 `route-config.jsx`에 연결된 `loginAction`이 실행됩니다.
  - `useActionData` 훅을 사용하여 `loginAction`에서 반환된 에러 메시지를 받아와 사용자에게 보여줍니다.

<!-- end list -->

```jsx
// frontend/src/components/auth/LoginForm.jsx

import {Form, Link, useActionData} from 'react-router-dom';
import styles from './LoginForm.module.scss';

const LoginForm = () => {

   // action 함수가 리턴한 데이터 가져오기
    const error = useActionData();

    return (
        <>
            <Form method="post" className={styles.form}>
                <h1>Log in</h1>
                {/* ... (input 필드) ... */}
                <div className={styles.actions}>
                    <button type="submit" className={styles.loginButton}>Login</button>
                </div>
                {error && <p className={styles.error}>{error}</p>}
                {/* ... */}
            </Form>
        </>
    );
};

export default LoginForm;
```

#### 2.2.3. `events-loader.js` & `route-config.jsx`: 전역 상태 관리 및 라우트 보호

  - **`userDataLoader`**:

      - 애플리케이션이 로드될 때 `localStorage`에서 `userData`를 읽어오는 **로더 함수**입니다.
      - `RootLayout`에 연결되어, 앱 전역에서 로그인 상태를 확인할 수 있는 데이터를 제공합니다.

  - **`useRouteLoaderData`**:

      - `RootLayout`의 `loader`가 반환한 데이터를 하위 컴포넌트(e.g., `WelcomePage`, `MainNavigation`)에서 쉽게 가져올 수 있도록 해주는 훅입니다.
      - 라우터 설정에서 `id`를 부여(`id: 'user-token-data'`)하여 특정 로더의 데이터를 식별합니다.

  - **`authCheckLoader` (라우트 가드)**:

      - 로그인이 필요한 페이지(`events` 경로)에 접근하기 전에 실행되는 로더입니다.
      - `isLoggedIn()` 함수로 로그인 상태를 확인하여, 로그인되어 있지 않으면 `redirect('/')`를 반환해 홈으로 쫓아냅니다. 이를 **라우트 가드(Route Guard)** 라고 합니다.

  - **`logoutAction`**:

      - 로그아웃을 처리하는 액션 함수입니다.
      - `localStorage`에서 `userData`를 제거합니다.
      - `redirect('/')`를 반환하여 홈으로 이동시킵니다.

<!-- end list -->

```javascript
// frontend/src/loader/events-loader.js (일부)

import { redirect } from 'react-router-dom';

// 토큰데이터를 파싱하는 함수
const parseUserData = () => JSON.parse(localStorage.getItem('userData'));

// 로컬 스토리지에 토큰 데이터를 불러오는 로더
export const userDataLoader = () => parseUserData();

const isLoggedIn = () => parseUserData() !== null;

// 로그인 여부를 검사하여 로그인하지 않았다면 로그인페이지로 돌려보내는 로더
export const authCheckLoader = () => {
    if (!isLoggedIn()) {
        alert('로그인이 필요한 서비스입니다.');
        return redirect('/');
    }
    return null; // 현재 페이지에 머물게 됨
};

// 로컬 스토리지에서 토큰값을 뽑아주는 함수
export const getUserToken = () => parseUserData()?.token;
```

```jsx
// frontend/src/routes/route-config.jsx (일부)

const router = createBrowserRouter([
    {
        path: '/',
        element: <RootLayout/>,
        errorElement: <ErrorPage/>,
        loader: userDataLoader, // 앱 전역에서 사용자 데이터 로드
        id: 'user-token-data', // 하위 컴포넌트에서 데이터 참조를 위한 ID
        children: [
            // ... (HomeLayout 및 하위 라우트)
            {
                path: 'events',
                element: <EventLayout/>,
                loader: authCheckLoader, // events 경로 접근 시 로그인 상태 확인
                children: [
                    // ... (events 하위 라우트)
                ]
            },
        ]
    },
]);
```

-----

## 3\. 이벤트 기능 및 인증 적용

### 3.1. 백엔드: 인증된 사용자만 이벤트 생성

#### 3.1.1. `SecurityConfig.java`: API 경로별 접근 제어

  - **역할**: Spring Security 설정을 통해 API 엔드포인트의 보안 규칙을 정의합니다.
  - **주요 설정**:
      - `csrf(csrf -> csrf.disable())`: CSRF(Cross-Site Request Forgery) 보호 기능을 비활성화합니다. (Stateless한 REST API에서는 보통 비활성화)
      - `sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))`: 세션을 사용하지 않는 **Stateless 인증 방식**을 설정합니다. JWT 인증에 필수적입니다.
      - `authorizeHttpRequests()`: 경로별 인가(Authorization) 규칙을 설정합니다.
          - `/api/auth/**`: `permitAll()`로 설정하여 로그인, 회원가입 등 인증 관련 API는 누구나 접근 가능하게 합니다.
          - `/api/**`: `authenticated()`로 설정하여 `/api/auth`를 제외한 모든 `/api` 경로는 **인증된 사용자만 접근**할 수 있도록 제한합니다.
      - `addFilterBefore(authenticationFilter, ...)`: 모든 요청이 컨트롤러에 도달하기 전에 `JwtAuthenticationFilter`가 먼저 실행되도록 설정합니다.

<!-- end list -->

```java
// backend/src/main/java/com/study/event/config/SecurityConfig.java

@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter authenticationFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .csrf(csrf -> csrf.disable())
                .cors(cors -> cors.configure(http))
                // 세션 인증 비활성화
                .sessionManagement(session ->
                        session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                )
                // 인가 설정
                .authorizeHttpRequests(auth ->
                        auth
                                // '/api/auth'로 시작하는 요청은 인증 필요 없음
                                .requestMatchers("/api/auth/**").permitAll()
                                // 그 외 '/api'로 시작하는 요청은 모두 인증 필수
                                .requestMatchers("/api/**").authenticated()
                                .anyRequest().permitAll()
                )
                // 직접 만든 토큰 검증 필터를 UsernamePasswordAuthenticationFilter 전에 배치
                .addFilterBefore(authenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

#### 3.1.2. `JwtAuthenticationFilter.java`: JWT 토큰 검증 필터

  - **역할**: 클라이언트의 모든 요청에 대해 JWT 토큰의 유효성을 검증하는 필터입니다.
  - **`doFilterInternal` 메서드**:
    1.  `resolveTokenFromHeader()`를 통해 요청 헤더(`Authorization: Bearer <token>`)에서 토큰을 추출합니다.
    2.  `validateAndAuthenticate()`를 호출하여 토큰이 유효한지 검사합니다.
    3.  토큰이 유효하면, 토큰에서 사용자 이메일을 추출합니다.
    4.  `SecurityContextHolder`에 인증된 사용자 정보(`Authentication` 객체)를 저장합니다. 이렇게 저장된 정보는 이후 컨트롤러 등에서 `@AuthenticationPrincipal` 어노테이션으로 쉽게 꺼내 쓸 수 있습니다.
    5.  인증 과정이 끝나면 `filterChain.doFilter()`를 호출하여 다음 필터 또는 컨트롤러로 요청을 전달합니다.

<!-- end list -->

```java
// backend/src/main/java/com/study/event/jwt/JwtAuthenticationFilter.java

@Component
@Slf4j
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider tokenProvider;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        // 1. 요청 헤더에서 토큰 추출
        String token = resolveTokenFromHeader(request);

        // 2. 토큰 유효성 검증 및 인증 정보 저장
        validateAndAuthenticate(token);

        // 3. 다음 필터로 요청 전달
        filterChain.doFilter(request, response);
    }

    private void validateAndAuthenticate(String token) {
        // 토큰이 존재하고 유효하면
        if (StringUtils.hasText(token) && tokenProvider.validateToken(token)) {
            // 토큰에서 로그인한 사용자의 이메일 추출
            String userEmail = tokenProvider.getCurrentLoginUserEmail(token);

            // Spring Security에게 접근을 허용하라고 명령
            Authentication authentication =
                    new UsernamePasswordAuthenticationToken(userEmail, null, new ArrayList<>());
            SecurityContextHolder.getContext().setAuthentication(authentication);

            log.info("authentication success: email - {}", userEmail);
        }
    }
    // ...
}
```

#### 3.1.3. `EventController.java`: 이벤트 생성 API 수정

  - `create` 메서드에 `@AuthenticationPrincipal String email` 파라미터를 추가합니다.
  - `@AuthenticationPrincipal` 어노테이션은 `JwtAuthenticationFilter`가 `SecurityContextHolder`에 저장해 둔 \*\*인증된 사용자의 정보(이메일)\*\*를 파라미터에 주입해줍니다.
  - 주입받은 `email`을 `eventService.saveEvent()` 메서드로 전달하여 어떤 사용자가 이벤트를 생성했는지 기록합니다.

<!-- end list -->

```java
// backend/src/main/java/com/study/event/api/EventController.java (일부)

@RestController
@RequestMapping("/api/events")
@Slf4j
@RequiredArgsConstructor
public class EventController {

    private final EventService eventService;

    // ...

    // 생성 요청
    @PostMapping
    public ResponseEntity<?> create(
            @RequestBody EventCreate dto,
            @AuthenticationPrincipal String email // 인증된 사용자의 이메일을 받음
    ) {
        eventService.saveEvent(dto, email);

        return ResponseEntity.ok().body(Map.of(
                "message", "이벤트가 정상 등록되었습니다."
        ));
    }
    // ...
}
```

#### 3.1.4. `EventService.java`: 이벤트 생성 로직에 사용자 정보 추가

  - `saveEvent` 메서드가 `email`을 추가로 받도록 수정되었습니다.
  - `getCurrentLoggedInUser()` 헬퍼 메서드를 통해 이메일로 `EventUser` 엔티티를 조회합니다.
  - 생성된 `Event` 엔티티에 `setEventUser()`를 통해 조회한 사용자 정보를 설정합니다.
  - 이렇게 함으로써 `Event`와 `EventUser` 간의 **연관관계**가 맺어집니다.

<!-- end list -->

```java
// backend/src/main/java/com/study/event/service/EventService.java (일부)

@Service
@Slf4j
@RequiredArgsConstructor
@Transactional
public class EventService {

    private final EventRepository eventRepository;
    private final EventUserRepository eventUserRepository;

    // ...

    // 이벤트 생성
    public void saveEvent(EventCreate dto, String email) {
        Event event = dto.toEntity();
        // 현재 로그인한 사용자 정보 조회
        EventUser foundUser = getCurrentLoggedInUser(email);
        // 이벤트 엔티티에 사용자 정보 설정
        event.setEventUser(foundUser);

        eventRepository.save(event);
    }

    // 로그인한 사용자의 엔터티정보를 불러오는 메서드
    private EventUser getCurrentLoggedInUser(String email) {
        return eventUserRepository.findByEmail(email).orElseThrow();
    }
}
```

### 3.2. 프론트엔드: 이벤트 생성 요청 시 토큰 전달

#### 3.2.1. `events-actions.js`: `saveAction` 수정

  - 이벤트 생성/수정 요청을 보내는 `saveAction` 함수를 수정합니다.
  - `fetch` 요청의 `headers`에 `Authorization` 헤더를 추가합니다.
  - 헤더 값으로는 `localStorage`에 저장된 토큰을 `getUserToken()` 함수로 가져와 ` Bearer  ` 접두사와 함께 전달합니다.
  - 이 토큰을 통해 백엔드 서버는 요청을 보낸 사용자를 식별하고 인가 처리를 할 수 있습니다.

<!-- end list -->

```javascript
// frontend/src/loader/events-actions.js (일부)

import { getUserToken } from './events-loader.js';

export const saveAction = async ({ request, params }) => {
    // ...
    const formData = await request.formData();
    const payload = { /* ... */ };
    let requestUrl = EVENT_API_URL;
    if (request.method === 'PUT') {
        requestUrl += `/${params.eventId}`;
    }

    const response = await fetch(requestUrl, {
        method: request.method,
        headers: {
            'Content-Type': 'application/json',
            // Authorization 헤더에 Bearer 토큰 추가
            'Authorization': 'Bearer '+ getUserToken()
        },
        body: JSON.stringify(payload)
    });
    // ...
    return redirect('/events');
};
```

-----

## 4\. 데이터베이스 및 기타 설정

### 4.1. `build.gradle`: P6Spy 의존성 추가

  - **P6Spy**: 실행되는 SQL 쿼리를 가로채서 파라미터가 바인딩된 **완성된 SQL문**을 로그로 보여주는 라이브러리입니다.
  - JPA나 MyBatis 사용 시, 로그에 `?`로 표시되는 파라미터 값을 직접 확인할 수 있어 디버깅에 매우 유용합니다.
  - `build.gradle`의 `dependencies` 블록에 아래 의존성을 추가하여 사용합니다.

<!-- end list -->

```gradle
// backend/build.gradle (일부)

dependencies {
    // ...
    // 쿼리파라미터 추가 외부로그 남기기
    implementation 'com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.9.1'
}
```

### 4.2. `P6SpyConfig.java`: P6Spy 로그 포맷 설정

  - **역할**: P6Spy가 출력하는 로그의 형식을 커스터마이징하는 설정 클래스입니다.
  - `MessageFormattingStrategy` 인터페이스를 구현하여 `formatMessage` 메서드를 오버라이드합니다.
  - `formatSql` 메서드 내부에서 Hibernate의 `FormatStyle`을 사용하여 SQL을 보기 좋게 정렬(Pretty Printing)하고, 최종 로그 메시지를 원하는 형식으로 조합하여 반환합니다.

<!-- end list -->

```java
// backend/src/main/java/com/study/event/config/P6SpyConfig.java

package com.study.event.config;

import com.p6spy.engine.logging.Category;
import com.p6spy.engine.spy.P6SpyOptions;
import com.p6spy.engine.spy.appender.MessageFormattingStrategy;
import jakarta.annotation.PostConstruct;
import org.hibernate.engine.jdbc.internal.FormatStyle;
import org.springframework.context.annotation.Configuration;

@Configuration
public class P6SpyConfig implements MessageFormattingStrategy {

    @PostConstruct
    public void setLogMessageFormat() {
        P6SpyOptions.getActiveInstance().setLogMessageFormat(this.getClass().getName());
    }

    @Override
    public String formatMessage(int connectionId, String now, long elapsed, String category, String prepared, String sql, String url) {
        sql = formatSql(category, sql);
        // [카테고리] | 실행시간 ms | SQL문 형식으로 로그 출력
        return String.format("[%s] | %d ms | %s", category, elapsed, sql);
    }

    private String formatSql(String category, String sql) {
        if (sql == null || sql.trim().isEmpty()) return sql;

        // DDL 또는 SQL문에 따라 포맷팅 적용
        if (Category.STATEMENT.getName().equals(category) || Category.BATCH.getName().equals(category)) {
            String trimmedSQL = sql.trim().toLowerCase();
            if (trimmedSQL.startsWith("create") || trimmedSQL.startsWith("alter") || trimmedSQL.startsWith("comment")) {
                sql = FormatStyle.DDL.getFormatter().format(sql);
            } else {
                sql = FormatStyle.BASIC.getFormatter().format(sql);
            }
            return sql;
        }
        return sql;
    }
}
```
