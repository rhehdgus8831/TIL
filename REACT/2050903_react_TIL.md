
# Event Fullstack Application 학습 노트

##  오늘의 학습 목표: 인증 및 인가 처리

오늘은 백엔드(Spring Boot)와 프론트엔드(React)를 연동하여 이메일 기반의 회원가입 및 인증 프로세스를 구현했습니다. 사용자가 입력한 이메일의 유효성과 중복을 검사하고, 인증 코드를 메일로 발송한 뒤, 사용자가 해당 코드를 정확히 입력했는지 검증하는 전체 흐름을 학습했습니다.

**주요 구현 기능:**
- **Backend**:
    - 이메일 중복 검사 API
    - Spring Mail을 이용한 인증 코드 발송
    - 인증 코드 검증 API
    - 관련 엔터티 및 데이터베이스 연동 처리
- **Frontend**:
    - 회원가입 UI/UX (단계별 폼)
    - 이메일 패턴 및 중복 실시간 검증
    - 인증 코드 입력 UI 및 타이머 기능
    - 서버와의 비동기 통신 처리

---

##  Backend (Spring Boot)

### 1. 프로젝트 설정 및 의존성

#### `build.gradle`
프로젝트의 의존성을 관리합니다. Spring Boot, Spring Data JPA, Lombok, MariaDB 드라이버 등 핵심 라이브러리 외에 **QueryDSL**과 **Spring Boot Starter Mail**을 추가하여 각각 동적 쿼리와 이메일 발송 기능을 구현했습니다.

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.5.5'
    id 'io.spring.dependency-management' version '1.1.7'
}

group = 'com.study'
version = '0.0.1-SNAPSHOT'
description = 'backend'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly 'org.projectlombok:lombok'
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    runtimeOnly 'org.mariadb.jdbc:mariadb-java-client'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'

    // QueryDSL 의존성 추가
    implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
    annotationProcessor 'com.querydsl:querydsl-apt:5.0.0:jakarta'
    annotationProcessor 'jakarta.annotation:jakarta.annotation-api'
    annotationProcessor 'jakarta.persistence:jakarta.persistence-api'

    // 이메일 전송 라이브러리
    implementation 'org.springframework.boot:spring-boot-starter-mail'
}

tasks.named('test') {
    useJUnitPlatform()
}
````

#### `application.yml`

서버 포트, 데이터베이스 연결 정보, JPA 설정과 함께 이메일 발송을 위한 **SMTP 서버 정보**를 설정합니다.

```yaml
server:
  port: 9000


spring:
  # email setting
  mail:
    host: smtp.naver.com
    port: 465
    username: * # github 유출 금지
    password: * # github 유출 금지
    protocol: smtp
    properties:
      mail:
        smtp:
          auth: true
          ssl:
            enable: true

  # database setting
  datasource:
    url: jdbc:mariadb://localhost:3306/eventdb
    username: root
    password: mariadb
    driver-class-name: org.mariadb.jdbc.Driver
  jpa:
    # DBMS dialect setting
    database-platform: org.hibernate.dialect.MariaDB106Dialect
    hibernate:
      # ddl
      ddl-auto: create
    properties:
      hibernate:
        format_sql: true # SQL log
    database: mysql

# log level setting
logging:
  level:
    root: info
    com.study.event: debug
    org.hibernate.SQL: debug
```

### 2\. 회원 및 인증 관련 엔터티

#### `EventUser.java`

회원 정보를 저장하는 엔터티입니다. 이메일 중복을 막기 위해 `unique = true` 속성을 사용했으며, 이메일 인증 완료 여부를 `emailVerified` 필드로 관리합니다.

```java
package com.study.event.domain.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.CreationTimestamp;

import java.time.LocalDateTime;

@Getter
@ToString
@EqualsAndHashCode
@NoArgsConstructor
@AllArgsConstructor
@Builder

@Entity
@Table(name = "tbl_event_user")
public class EventUser {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "ev_user_id")
    private Long id;

    @Column(name = "ev_user_email", nullable = false, unique = true)
    private String email;

    // NN을 안거는 이유 : SNS 로그인한 회원, 인증번호만 받고 회원가입을 마무리하지 않은 회원 때문
    @Column(length = 500)
    private String password;

    @Enumerated(EnumType.STRING)
    @Builder.Default
    private Role role = Role.COMMON; // 권한

    //    @CreationTimestamp
    private LocalDateTime createdAt;

    // 이메일 인증을 완료했는지 여부
    @Column(nullable = false)
    private boolean emailVerified;

    // 이메일 인증을 완료하는 헬퍼함수
    public void completeVerifying() {
        this.emailVerified = true;
    }
}
```

#### `EmailVerification.java`

발급된 인증 코드와 만료 시간을 관리하는 엔터티입니다. `EventUser`와 **1:1 연관관계**를 맺어 어떤 회원의 인증 정보인지 식별합니다.

```java
package com.study.event.domain.entity;

import jakarta.persistence.*;
import lombok.*;

import java.time.LocalDateTime;

@Getter
@ToString
@EqualsAndHashCode
@NoArgsConstructor
@AllArgsConstructor
@Builder

@Entity
@Table(name = "tbl_email_verification")
public class EmailVerification {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "verification_id")
    private Long id;

    @Column(nullable = false)
    private String verificationCode; // 인증코드

    @Column(nullable = false)
    private LocalDateTime expiryDate; // 인증 만료시간

    // 연관관계 설정
    @OneToOne
    @JoinColumn(name = "event_user_id", referencedColumnName = "ev_user_id")
    private EventUser eventUser;

    public void updateNewCode(String newCode) {
        this.verificationCode = newCode;
        this.expiryDate = LocalDateTime.now().plusMinutes(5);
    }
}
```

### 3\. Repository

#### `EventUserRepository.java`

이메일 존재 여부를 확인하는 `existsByEmail`과 이메일로 회원을 찾는 `findByEmail` 메서드를 정의했습니다.

```java
package com.study.event.repository;

import com.study.event.domain.entity.EventUser;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface EventUserRepository extends JpaRepository<EventUser, Long> {

    boolean existsByEmail(String email);
    Optional<EventUser> findByEmail(String email);

}
```

#### `EmailVerificationRepository.java`

`EventUser` 엔터티를 통해 인증 정보를 조회하는 `findByEventUser` 메서드를 정의했습니다.

```java
package com.study.event.repository;

import com.study.event.domain.entity.EmailVerification;
import com.study.event.domain.entity.EventUser;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface EmailVerificationRepository  extends JpaRepository<EmailVerification, Long> {

    // 사용자의 ID를 통해 인증코드 정보 조회
    Optional<EmailVerification> findByEventUser(EventUser eventUser);
}
```

### 4\. Service

#### `EventUserService.java`

회원 인증 로직의 핵심입니다. 이메일 중복 확인, 인증 코드 생성 및 메일 발송, 코드 검증 등 주요 비즈니스 로직을 담당합니다.

  - **`checkEmailDuplicate(String email)`**: 이메일이 중복되는지 확인하고, 중복되지 않으면 임시 회원 가입 및 인증 메일 발송 프로세스(`processSignup`)를 시작합니다.
  - **`sendVerificationEmail(String email)`**: `JavaMailSender`를 이용해 실제 이메일을 발송합니다. 인증 코드는 `generateCode()`를 통해 생성됩니다.
  - **`isMatchCode(String email, String code)`**: 사용자가 입력한 코드가 DB에 저장된 코드와 일치하는지, 그리고 만료 시간이 지나지 않았는지 확인합니다. 인증에 성공하면 `emailVerified` 상태를 `true`로 변경하고, 실패하면 `updateVerificationCode`를 통해 새 코드를 발송합니다.

<!-- end list -->

```java
package com.study.event.service;

import com.study.event.domain.entity.EmailVerification;
import com.study.event.domain.entity.EventUser;
import com.study.event.repository.EmailVerificationRepository;
import com.study.event.repository.EventUserRepository;
import jakarta.mail.internet.MimeMessage;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;

@Service
@Slf4j
@RequiredArgsConstructor
@Transactional
public class EventUserService {

    // 메일 발송인의 정보
    @Value("${spring.mail.username}")
    private String mailHost;

    // 이메일 발송을 위한 의존객체
    private final JavaMailSender mailSender;

    private final EventUserRepository eventUserRepository;
    private final EmailVerificationRepository emailVerificationRepository;

    // 이메일 중복확인 처리
    @Transactional(readOnly = true)
    public boolean checkEmailDuplicate(String email) {

        // 중복확인
        boolean flag = eventUserRepository.existsByEmail(email);
        log.info("Checking email {} is duplicate: {}", email, flag);

        // 사용가능한 이메일인 경우 인증메일 발송
        if (!flag) processSignup(email);

        return flag;
    }

    // 인증 코드를 발송할 때 사용할 임시 회원가입 로직
    // 인증코드를 데이터베이스에 저장하려면 회원정보가 필요
    private void processSignup(String email) {
        // 1. 임시 회원가입
        EventUser tempUser = EventUser.builder()
                .email(email)
                .build();

        EventUser savedUser = eventUserRepository.save(tempUser);

        // 2. 인증 메일 발송
        String code = sendVerificationEmail(email);

        // 3. 인증 코드와 만료시간을 DB에 저장
        EmailVerification verification = EmailVerification.builder()
                .verificationCode(code)
                .expiryDate(LocalDateTime.now().plusMinutes(5)) // 만료시간 5분 설정
                .eventUser(savedUser) // FK 설정
                .build();
        emailVerificationRepository.save(verification);
    }

    // 이메일 인증코드 발송 로직
    private String sendVerificationEmail(String email) {

        // 인증코드 생성
        String code = generateCode();

        // 메일 전송 로직
        MimeMessage mimeMessage = mailSender.createMimeMessage();

        try {
            MimeMessageHelper messageHelper
                    = new MimeMessageHelper(mimeMessage, false, "UTF-8");

            // 누구에게 이메일을 보낼지
            messageHelper.setTo(email);

            // 누가 보내는 건지
            messageHelper.setFrom(mailHost);

            // 이메일 제목 설정
            messageHelper.setSubject("[인증메일] 중앙정보스터디 가입 인증 메일입니다.");
            // 이메일 내용 설정
            messageHelper.setText(
                    "인증 코드: <b style=\"font-weight: 700; letter-spacing: 5px; font-size: 30px;\">" + code + "</b>"
                    , true
            );

            // 메일 보내기
            mailSender.send(mimeMessage);

            log.info("{} 님에게 이메일이 발송되었습니다.", email);
            return code;

        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException("메일 발송에 실패했습니다.");
        }

    }

    // 무작위로 1000~9999 사이의 랜덤 숫자를 생성
    private String generateCode() {
        return String.valueOf((int) (Math.random() * 9000) + 1000);
    }


    /**
     * 클라이언트가 전송한 인증코드를 검증하는 처리
     */
    public boolean isMatchCode(String email, String code) {

        // 이메일을 통해 사용자의 PK를 조회
        EventUser eventUser = eventUserRepository.findByEmail(email).orElseThrow();

        // 사용자의 인증코드를 FK를 통해 데이터베이스에서 조회
        EmailVerification verification
                = emailVerificationRepository.findByEventUser(eventUser).orElseThrow();

        // 코드가 일치하고 만료시간이 지나지 않았는지 체크
        if (
                code.equals(verification.getVerificationCode())
                        && verification.getExpiryDate().isAfter(LocalDateTime.now())
        ) {
            // 이메일 인증 완료처리
            eventUser.completeVerifying();
            eventUserRepository.save(eventUser);

            // 인증번호를 데이터베이스에서 삭제
            emailVerificationRepository.delete(verification);

            return true;
        }
        // 인증코드가 틀렸거나 만료된 경우 자동으로 인증코드를 재발송
        updateVerificationCode(email, verification);
        return false;
    }

    // 인증코드 재발급 처리
    private void updateVerificationCode(String email, EmailVerification verification) {

        // 1. 새인증코드를 생성하고 메일을 재발송
        String newCode = sendVerificationEmail(email);

        // 2. 데이터베이스에 인증코드와 만료시간을 갱신
        verification.updateNewCode(newCode);
        emailVerificationRepository.save(verification);
    }
}
```

### 5\. Controller

#### `AuthController.java`

프론트엔드와 직접 통신하는 API 엔드포인트입니다.

  - **`/api/auth/check-email`**: 이메일 중복 확인 요청을 처리합니다.
  - **`/api/auth/code`**: 인증 코드 검증 요청을 처리합니다.

<!-- end list -->

```java
package com.study.event.api;

import com.study.event.service.EventUserService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
@Slf4j
@RequiredArgsConstructor
@RequestMapping("/api/auth")
public class AuthController {

    private final EventUserService eventUserService;

    // email 중복확인 API 생성
    @GetMapping("/check-email")
    public ResponseEntity<?> checkEmail(String email) {
        boolean isDuplicate = eventUserService.checkEmailDuplicate(email);
        String message = isDuplicate ? "이메일이 중복되었습니다." : "사용 가능한 이메일입니다.";

        return ResponseEntity.ok().body(Map.of(
                "isDuplicate", isDuplicate,
                "message", message
        ));
    }

    // 인증 코드 검증 API
    @GetMapping("/code")
    public ResponseEntity<?> verifyCode(String email, String code) {
        log.info("{}'s verify code is [ {} ]", email, code);

        boolean isMatch = eventUserService.isMatchCode(email, code);

        log.info("code matches? - {}", isMatch);

        return ResponseEntity.ok().body(Map.of(
                "isMatch", isMatch
        ));
    }
}
```

-----

##  Frontend (React)

### 1\. API 연동 설정

#### `src/config/host-config.js`

백엔드 서버의 주소를 변수로 관리하여 개발 환경과 배포 환경에 유연하게 대응할 수 있도록 설정합니다.

```javascript
// 백엔드 로컬서버의 포트번호
const LOCAL_PORT = 9000;

const clientHostName = window.location.hostname;

// 백엔드 호스트를 동적 결정
let backendHostName;

if (clientHostName === 'localhost') {
    backendHostName = `http://localhost:${LOCAL_PORT}`;
} else if (clientHostName === 'strawberry.com') {
    backendHostName = `https://api.berry.com`;
}

// 기본 API 엔드포인트 저장
const API_BASE_ENDPOINT = `${backendHostName}/api`;

const EVENT = '/events';
const AUTH = '/auth';

// http://localhost:9000/api/events
export const EVENT_API_URL = `${API_BASE_ENDPOINT}${EVENT}`;
// http://localhost:9000/api/auth
export const AUTH_API_URL = `${API_BASE_ENDPOINT}${AUTH}`;
```

### 2\. 회원가입 UI 및 로직

#### `src/components/auth/SignUpForm.jsx`

회원가입 절차를 단계별로 관리하는 컨테이너 컴포넌트입니다. `step` 상태를 통해 현재 단계를 제어하며, 각 단계에 맞는 컴포넌트(`EmailInput`, `VerificationInput`)를 렌더링합니다.

```javascript
import styles from './SignUpForm.module.scss';
import EmailInput from './EmailInput.jsx';
import VerificationInput from './verificationInput.jsx';
import {useState} from 'react';
import ProgressBar from '../common/ProgressBar.jsx';

const SignUpForm = () => {

    const [enteredEmail, setEnteredEmail] = useState();


    // 현재 어떤 스텝인지 확인
    const [step, setStep] = useState(1);
    // 프로그레스바 노출 여부
    const [isNext, setIsNext] = useState(false);

    // 이메일 중복확인이 끝날때 호출될 함수
    const emailSuccessHandler = (email) => {

        setIsNext(true); // progress bar 띄우기

        setTimeout(() => {
            setStep(2);
            setIsNext(false);
            setEnteredEmail(email);
        }, 1000);

    };

    return (
        <div className={styles.signupForm}>
            <div className={styles.formStepActive}>
                {step === 1 && <EmailInput onSuccess={emailSuccessHandler}/>}
                {step === 2 && <VerificationInput email={enteredEmail}/>}

                {isNext && <ProgressBar/>}

            </div>
        </div>
    );
};

export default SignUpForm;
```

#### `src/components/auth/EmailInput.jsx`

회원가입 1단계 컴포넌트입니다. 사용자가 이메일을 입력하면 **이메일 형식 유효성 검사**와 **중복 확인**을 수행합니다. `lodash`의 `debounce` 함수를 사용하여 API 요청 횟수를 최적화했습니다.

```javascript
import {useEffect, useRef, useState} from 'react';
import styles from './SignUpForm.module.scss';
import {AUTH_API_URL} from '../../config/host-config.js';
import {debounce} from 'lodash';

const EmailInput = ({onSuccess}) => {

    const emailRef = useRef();

    // 에러 상태 메세지를 관리
    const [error, setError] = useState('');


    // 화면이 랜더링되자마자 입력창에 포커싱
    useEffect(() => {
        emailRef.current.focus();
    }, []);

    // 이메일 입력 이벤트
    const handleEmail = e => {
        const inputValue = e.target.value;

        // 이메일 패턴 검증
        const emailPattern = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        if (!emailPattern.test(inputValue)) {
            setError('이메일이 올바르지 않습니다.');
            return;
        }

        // 이메일 중복확인 검증
        (async () => {
            const response = await fetch(`${AUTH_API_URL}/check-email?email=${inputValue}`);
            const {isDuplicate, message} = await response.json();
            if (isDuplicate) {
                setError(message);
            } else {
                onSuccess(inputValue); // 성공 시 다음 단계로 진행
            }
        })();

        setError('');
    };


    return (
        <>
            <p>Step 1: 유효한 이메일을 입력해주세요.</p>
            <input
                ref={emailRef}
                className={error ? styles.invalidInput : ''}
                type="email"
                placeholder="Enter your email"
                onChange={debounce(handleEmail,1000)}
            />
            {error && <p className={styles.errorMessage}>{error}</p>}
        </>
    );
};

export default EmailInput;
```

#### `src/components/auth/verificationInput.jsx`

회원가입 2단계 컴포넌트로, 4자리 인증 코드를 입력받습니다.

  - **다중 `ref` 관리**: `useRef`를 배열로 사용하여 4개의 입력 필드를 제어하고, 한 글자 입력 시 다음 칸으로 자동 포커스 이동 기능을 구현했습니다.
  - **타이머**: `useEffect`와 `setInterval`을 이용해 5분 타이머를 구현하여 사용자에게 남은 시간을 보여줍니다.
  - **서버 검증**: 4자리 코드가 모두 입력되면 `debounce`를 적용한 `fetchVerifying` 함수를 호출하여 서버에 코드를 전송하고 유효성을 검증합니다.

<!-- end list -->

```javascript
import styles from './SignUpForm.module.scss';
import {useEffect, useRef, useState} from 'react';
import {debounce} from 'lodash';
import {AUTH_API_URL} from '../../config/host-config.js';

const VerificationInput = ({email}) => {

    // 완성된 인증코드를 상태관리
    const [codes, setCodes] = useState(['', '', '', '']);

    // 에러 메세지 상태 관리
    const [error, setError] = useState('');

    // 인증 코드 만료 타이머 시간을 상태관리
    const [timer, setTimer] = useState(300);

    // ref를 배열로 관리하는 법
    const inputRefs = useRef([]);

    // 수동으로 ref배열에 input태그들 저장하기
    const bindRef = ($input, index) => {
        inputRefs.current[index] = $input;
    };

    const focusNextInput = index => {
        if (index < inputRefs.current.length) {
            inputRefs.current[index].focus();
        } else {
            inputRefs.current[index - 1].blur();
        }
    };

    // 서버에 인증코드 전송
    const fetchVerifying = debounce(async (verifyCode) => {
        const response
            = await fetch(`${AUTH_API_URL}/code?email=${email}&code=${verifyCode}`);
        const {isMatch} = await response.json();

        // 검증에 실패했을 경우
        if (!isMatch){
            setTimer(300);
            setError('유효하지 않거나 만료된 인증코드입니다. 인증코드를 재발송합니다.')
            setCodes(Array(4).fill(''));
            inputRefs.current[0].focus();
            return;
        }
        // 검증 성공 시
        setError('');

    }, 1000);


    // 숫자 입력 이벤트
    const handleNumber = (index, e) => {
        const inputValue = e.target.value;
        if (inputValue !== '' && !/^\d$/.test(inputValue)) {
            return;
        }

        const copyCodes = [...codes];
        copyCodes[index] = inputValue;
        setCodes(copyCodes);
        focusNextInput(index + 1);

        // 모든 인증코드를 입력했을 때 서버에 인증코드를 전송
        if (copyCodes.every(code => code !== '')) {
            const verifyCode = copyCodes.join('');
            fetchVerifying(verifyCode);
        }
    };

    // 초기 랜더링 시 타이머를 가동
    useEffect(() => {
        const id = setInterval(() => {
            setTimer(prev => prev > 0 ? prev - 1 : 0);
        }, 1000);

        inputRefs.current[0].focus();

        return () => clearInterval(id);
    }, []);

    return (
        <>
            <p>Step 2: 이메일로 전송된 인증번호 4자리를 입력해주세요.</p>
            <div className={styles.codeInputContainer}>
                {
                    Array.from(new Array(4)).map((_, index) => (
                        <input
                            ref={($input) => bindRef($input, index)}
                            key={index}
                            type="text"
                            className={styles.codeInput}
                            maxLength={1}
                            onChange={(e) => handleNumber(index, e)}
                            value={codes[index]}
                        />
                    ))
                }
            </div>
            <div className={styles.timer}>
                {`${'0' + Math.floor(timer / 60)}:${('0' + (timer % 60)).slice(-2)}`}
            </div>
            {error && <p className={styles.errorMessage}>{error}</p>}
        </>
    );
};

export default VerificationInput;
```

### 3\. 라우팅 설정

#### `src/routes/route-config.jsx`

React Router를 사용하여 페이지 라우팅을 설정합니다. 회원가입 페이지(`/sign-up`)와 기본 페이지(`/`)를 정의하고, 각 경로에 맞는 레이아웃과 컴포넌트를 연결합니다.

```javascript
import {createBrowserRouter} from 'react-router-dom';
import ErrorPage from '../pages/ErrorPage.jsx';
import EventPage from '../pages/EventPage.jsx';
import RootLayout from '../layouts/RootLayout.jsx';
// ... (다른 import문 생략)
import HomeLayout from '../layouts/HomeLayout.jsx';
import WelcomePage from '../pages/WelcomePage.jsx';
import SignUpPage from '../pages/SignUpPage.jsx';

const router = createBrowserRouter([
    {
        path: '/',
        element: <RootLayout/>,
        errorElement: <ErrorPage/>,
        children: [
            {
                path: '',
                element: <HomeLayout />,
                children: [
                    {
                        index: true,
                        element: <WelcomePage />
                    },
                    {
                        path: '/sign-up',
                        element: <SignUpPage />
                    }
                ]
            },
            // ... (events 라우트 생략)
        ]
    },
]);

export default router;
```

-----

## ✨ 커밋 내역으로 본 개발 흐름

1.  **[BE] 회원 엔터티 및 리포지토리 생성**: `EventUser`, `EmailVerification` 엔터티와 관련 Repository를 먼저 설계하여 데이터베이스 구조를 잡았습니다.
2.  **[BE] 이메일 중복확인 API**: `/api/auth/check-email` 엔드포인트를 만들어 프론트에서 이메일 중복을 확인할 수 있도록 했습니다.
3.  **[BE] 이메일 인증메일 발송 로직**: `JavaMailSender`를 활용해 실제 이메일 발송 기능을 구현했습니다.
4.  **[BE] 이메일 인증메일 DB 처리**: 발송된 인증 코드를 DB에 저장하여 추후 검증할 수 있도록 했습니다.
5.  **[FE] 인증 관련 컴포넌트 생성**: `SignUpForm`, `EmailInput` 등 회원가입 UI를 구성하는 컴포넌트를 만들었습니다.
6.  **[FE] 이메일 패턴 및 중복 검증**: 사용자가 입력하는 이메일의 유효성을 실시간으로 검사하고, `debounce`를 적용하여 백엔드에 중복 확인 요청을 보냈습니다.
7.  **[FE] 인증번호 입력 페이지로 넘어가기**: 이메일 검증이 성공하면 다음 단계인 인증 코드 입력 화면으로 전환하는 로직을 구현했습니다.
8.  **[FE] 다중 ref 바인딩 및 입력칸 자동 이동**: 인증 코드 입력칸의 사용성을 높이기 위해 `useRef` 배열과 `focus` 이벤트를 활용했습니다.
9.  **[BE] 인증 코드 검증 API**: `/api/auth/code` 엔드포인트를 만들어 프론트에서 보낸 인증 코드를 검증하는 로직을 완성했습니다.
10. **[FE] 입력한 인증코드 서버 발송 및 타이머**: 4자리 코드를 모두 입력하면 서버에 검증을 요청하고, 타이머 UI를 통해 사용자 경험을 개선했습니다.

<!-- end list -->

```
```
