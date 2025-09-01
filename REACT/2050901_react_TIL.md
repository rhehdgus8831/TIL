# Event Fullstack Application 학습 노트

## Ⅰ. 프로젝트 개요

Spring Boot와 React를 사용하여 백엔드와 프론트엔드를 분리한 Fullstack 이벤트 관리 애플리케이션을 구축합니다. 이 과정을 통해 REST API 설계 및 구현, React-Router를 활용한 SPA(Single Page Application) 개발, 그리고 `fetch` API를 이용한 비동기 통신 방법을 복습하고, `react-router-dom`의 `loader` 기능을 심도 있게 학습합니다.

- **Backend**: Spring Boot, Spring Data JPA, MariaDB
- **Frontend**: React, Vite, react-router-dom, SASS

-----

## Ⅱ. Backend (Spring Boot)

### 1\. 프로젝트 설정 및 의존성

`build.gradle` 파일을 통해 프로젝트에 필요한 라이브러리들을 관리합니다.

```groovy
// build.gradle
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

// ... (생략)

dependencies {
    // Spring Data JPA: 데이터베이스 연동을 쉽게 해주는 라이브러리
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    // Spring Web: RESTful API 등 웹 개발에 필요한 기능 제공
    implementation 'org.springframework.boot:spring-boot-starter-web'
    // Lombok: 반복적인 코드를 어노테이션으로 자동 생성 (Getter, Setter, 생성자 등)
    compileOnly 'org.projectlombok:lombok'
    // Spring Boot DevTools: 개발 편의 기능 제공 (자동 재시작 등)
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    // MariaDB JDBC Driver: MariaDB 데이터베이스와 연동하기 위한 드라이버
    runtimeOnly 'org.mariadb.jdbc:mariadb-java-client'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

// ... (생략)
```

### 2\. `application.yml` - 핵심 설정

애플리케이션의 주요 설정을 담당합니다. 서버 포트, 데이터베이스 연결 정보, JPA 및 로깅 설정을 정의합니다.

```yaml
# application.yml
server:
  port: 9000 # 서버 포트를 9000번으로 설정

# database setting
spring:
  datasource:
    url: jdbc:mariadb://localhost:3306/eventdb
    username: root
    password: mariadb
    driver-class-name: org.mariadb.jdbc.Driver
  jpa:
    # DBMS dialect setting
    database-platform: org.hibernate.dialect.MariaDB106Dialect
    hibernate:
      # ddl-auto: update -> Entity 변경 시 DB 테이블 자동 업데이트
      ddl-auto: update
    properties:
      hibernate:
        format_sql: true # JPA가 생성하는 SQL을 보기 좋게 포맷팅
    database: mysql

# log level setting
logging:
  level:
    root: info
    com.study.event: debug
    org.hibernate.SQL: debug # 실행되는 SQL 쿼리를 로그로 출력
```

### 3\. CORS 전역 설정

프론트엔드(ex: `http://localhost:5173`)와 백엔드(`http://localhost:9000`)는 서로 다른 출처(Origin)를 가지므로, 브라우저의 동일 출처 정책(Same-Origin Policy)에 의해 통신이 차단될 수 있습니다. 이를 해결하기 위해 CORS(Cross-Origin Resource Sharing) 설정을 추가하여 특정 출처의 요청을 허용해야 합니다.

```java
// config/CorsConfig.java
package com.study.event.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

// 전역 크로스 오리진 설정 : 허용할 클라이언트 지정
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    // 허용할 프론트엔드 서버의 주소 목록
    private String[] permitUrls = {
            "http://localhost:5173",
            "http://localhost:5174",
            "http://localhost:5175",
            "http://localhost:5176",
            "http://localhost:5179",

    };

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry
                .addMapping("/api/**") // /api로 시작하는 모든 요청에 대해
                .allowedOrigins(permitUrls) // permitUrls에 명시된 출처의 요청을 허용
                .allowedMethods("*") // 모든 HTTP 메서드(GET, POST, PUT, DELETE 등) 허용
                .allowedHeaders("*") // 모든 헤더 허용
                .allowCredentials(true) // 보안 쿠키 허용
        ;
    }
}
```

### 4\. API 서버 상태 확인 (Health Check)

API 서버가 정상적으로 동작하는지 간단히 확인할 수 있는 엔드포인트를 제공하는 것은 좋은 습관입니다.

```java
// healthcheck/HealthCheckController.java
package com.study.event.healthcheck;

import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.time.LocalDateTime;
import java.util.Map;

// API가 살아있는지 확인하는 경로
@RestController
@Slf4j
public class HealthCheckController {

    @GetMapping("/status")
    public ResponseEntity<?> healthCheck() {

        return ResponseEntity.ok(
                Map.of(
                        "healthy", true,
                        "timestamp", LocalDateTime.now()
                )
        );
    }
}
```

### 5\. 도메인 설계 (Entity, DTO, Repository)

#### Event (Entity)

데이터베이스의 `tbl_event` 테이블과 매핑되는 핵심 객체입니다.

```java
// domain/entity/Event.java
package com.study.event.domain.entity;

// ... (import 생략)

@Getter @ToString @EqualsAndHashCode(of = "id")
@NoArgsConstructor @AllArgsConstructor
@Builder
@Entity
@Table(name = "tbl_event")
public class Event {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "ev_id")
    private Long id;

    @Column(name = "ev_title", nullable = false, length = 50)
    private String title; // 이벤트 제목

    @Column(name = "ev_desc")
    private String description; // 이벤트 설명

    @Column(name = "ev_image_path")
    private String image; // 이벤트 메인 이미지 경로

    @Column(name = "ev_start_date")
    private LocalDate date; // 이벤트 행사 시작 날짜

    @CreationTimestamp
    private LocalDateTime createdAt; // 이벤트 등록 날짜

    // 수정용 편의 메서드
    public void changeEvent(EventCreate dto) {
        this.title = dto.title();
        this.description = dto.desc();
        this.image = dto.imageUrl();
        this.date = dto.beginDate();
    }
}
```

#### DTO (Data Transfer Object)

계층 간 데이터 전송을 위해 사용하는 객체입니다. Entity를 직접 노출하지 않고 필요한 데이터만 담아 클라이언트와 통신함으로써 안정성을 높입니다. `record` 타입을 사용하면 불변 객체를 간결하게 정의할 수 있습니다.

**요청(Request) DTO**

```java
// domain/dto/request/EventCreate.java
public record EventCreate(
        String title,
        String desc,
        String imageUrl,
        @JsonFormat(pattern = "yyyy-MM-dd") // JSON 날짜 형식을 지정
        LocalDate beginDate
) {
    // DTO를 Entity로 변환하는 편의 메서드
    public Event toEntity() {
        return Event.builder()
                .title(this.title)
                .description(this.desc)
                .image(this.imageUrl)
                .date(this.beginDate)
                .build();
    }
}
```

**응답(Response) DTO**

```java
// domain/dto/response/EventResponse.java (목록 조회용)
@Builder
public record EventResponse(
        String eventId,
        String title,
        @JsonFormat(pattern = "yyyy년 MM월 dd일")
        LocalDate startDate,
        String imgUrl
) {
    // Entity를 DTO로 변환하는 편의 메서드
    public static EventResponse from(Event event) {
        return EventResponse.builder()
                .eventId(event.getId().toString())
                .imgUrl(event.getImage())
                .title(event.getTitle())
                .startDate(event.getDate())
                .build();
    }
}
```

```java
// domain/dto/response/EventDetailResponse.java (상세 조회용)
@Builder
public record EventDetailResponse(
        String id,
        String title,
        String desc,
        @JsonProperty("img-url") // JSON key 이름을 'img-url'로 변경
        String image,
        @JsonProperty("start-date")
        @JsonFormat(pattern = "yyyy년 MM월 dd일")
        LocalDate startDate
) {
    public static EventDetailResponse from(Event event) {
        // ... (builder 생략)
    }
}
```

#### EventRepository

`JpaRepository`를 상속받아 기본적인 CRUD 메서드를 자동으로 제공받습니다.

```java
// repository/EventRepository.java
package com.study.event.repository;

import com.study.event.domain.entity.Event;
import org.springframework.data.jpa.repository.JpaRepository;

public interface EventRepository extends JpaRepository<Event, Long> {
}
```

### 6\. 비즈니스 로직 (Service)

실질적인 비즈니스 로직을 처리하는 계층입니다. `@Transactional`을 통해 트랜잭션 관리를 합니다.

```java
// service/EventService.java
@Service
@Slf4j
@RequiredArgsConstructor
@Transactional
public class EventService {

    private final EventRepository eventRepository;

    // 전체 조회
    @Transactional(readOnly = true) // 조회 성능 최적화
    public List<EventResponse> getEvents() {
        return eventRepository.findAll()
                .stream()
                .map(EventResponse::from)
                .collect(Collectors.toList());
    }

    // 이벤트 생성
    public void saveEvent(EventCreate dto) {
        eventRepository.save(dto.toEntity());
    }

    // 이벤트 단일 조회
    @Transactional(readOnly = true)
    public EventDetailResponse findOne(Long id) {
        Event event = eventRepository.findById(id).orElseThrow();
        return EventDetailResponse.from(event);
    }

    // 이벤트 삭제
    public void deleteEvent(Long id) {
        eventRepository.deleteById(id);
    }

    // 이벤트 수정
    public void modifyEvent(EventCreate dto, Long id) {
        Event event = eventRepository.findById(id).orElseThrow();
        event.changeEvent(dto); // Entity의 변경 감지(dirty check)로 자동 업데이트
    }
}
```

### 7\. API 엔드포인트 (Controller)

HTTP 요청을 받아 해당 요청에 맞는 Service 메서드를 호출하고, 그 결과를 클라이언트에 응답합니다.

```java
// api/EventController.java
@RestController
@RequestMapping("/api/events")
@Slf4j
@RequiredArgsConstructor
public class EventController {

    private final EventService eventService;

    // 전체 조회 요청: GET /api/events
    @GetMapping
    public ResponseEntity<?> getList() {
        List<EventResponse> events = eventService.getEvents();
        return ResponseEntity.ok().body(events);
    }

    // 생성 요청: POST /api/events
    @PostMapping
    public ResponseEntity<?> create(@RequestBody EventCreate dto) {
        eventService.saveEvent(dto);
        return ResponseEntity.ok().body(Map.of("message", "이벤트가 정상 등록되었습니다."));
    }

    // 단일 조회 요청: GET /api/events/{eventId}
    @GetMapping("/{eventId}")
    public ResponseEntity<?> getEvent(@PathVariable Long eventId) {
        // ... (유효성 검사 생략)
        EventDetailResponse detailResponse = eventService.findOne(eventId);
        return ResponseEntity.ok().body(detailResponse);
    }

    // 삭제 요청: DELETE /api/events/{id}
    @DeleteMapping("/{id}")
    public ResponseEntity<?> deleteEvent(@PathVariable Long id) {
        eventService.deleteEvent(id);
        return ResponseEntity.ok().body(Map.of("message", "이벤트가 삭제되었습니다. id - " + id));
    }
    
    // 수정 요청: PUT /api/events/{id}
    @PutMapping("/{id}")
    public ResponseEntity<?> updateEvent(@PathVariable Long id, @RequestBody EventCreate dto) {
        eventService.modifyEvent(dto, id);
        return ResponseEntity.ok().body(Map.of("message", "이벤트가 수정되었습니다. id - " + id));
    }
}
```

-----

## Ⅲ. Frontend (React)

### 1\. 프로젝트 설정 및 라우팅

`Vite`를 사용하여 React 프로젝트를 구성하고, `react-router-dom`으로 페이지 간의 라우팅을 관리합니다.

#### 라우터 설정 (`route-config.jsx`)

`createBrowserRouter`를 사용하여 라우트 객체 배열을 정의합니다. 중첩 라우팅과 `Layout` 컴포넌트를 활용하여 공통 UI 구조를 관리합니다.

```jsx
// src/routes/route-config.jsx
import {createBrowserRouter} from 'react-router-dom';
// ... (Component Imports)
import { eventsListLoader, eventDetailLoader } from '../loader/events-loader.js';


const router = createBrowserRouter([
    {
        path: '/',
        element: <RootLayout/>, // 최상위 레이아웃
        errorElement: <ErrorPage/>,
        children: [
            { index: true, element: <HomePage/> },
            {
                path: 'events',
                element: <EventLayout/>, // 이벤트 관련 페이지들의 공통 레이아웃
                children: [
                    {
                        index: true,
                        element: <EventPage/>,
                        // loader: 페이지 렌더링 전에 실행되어 데이터를 미리 가져옴
                        loader: eventsListLoader
                    },
                    {
                        path: 'new',
                        element: <NewEventPage/>,
                    },
                    {
                        path: ':eventId', // 동적 경로 파라미터
                        element: <EventDetailPage/>,
                        loader : eventDetailLoader
                    }
                ]
            },
        ]
    },
]);

export default router;
```

#### `App.jsx`와 `main.jsx`

`RouterProvider` 컴포넌트에 생성한 라우터 설정을 props로 전달하여 애플리케이션에 라우팅 기능을 적용합니다.

```jsx
// src/App.jsx
import {RouterProvider} from 'react-router-dom';
import router from './routes/route-config.jsx';
import './App.css';

const App = () => (
    <RouterProvider router={router} />
);

export default App;
```

### 2\. 레이아웃 컴포넌트

`Outlet` 컴포넌트를 사용하여 중첩된 자식 라우트 컴포넌트가 렌더링될 위치를 지정합니다.

```jsx
// src/layouts/RootLayout.jsx
import React from 'react';
import MainNavigation from '../components/MainNavigation.jsx';
import {Outlet} from 'react-router-dom';

const RootLayout = () => {
    return (
        <>
         <MainNavigation/>
         <main>
             {/* 자식 라우트 컴포넌트가 이곳에 렌더링됩니다. */}
             <Outlet/>
         </main>
        </>
    );
};
export default RootLayout;
```

### 3\. **(핵심)** `loader`와 `useLoaderData`를 이용한 데이터 페칭

`useEffect`를 사용한 기존의 데이터 페칭 방식은 컴포넌트가 렌더링된 후 데이터를 요청하기 때문에 로딩 상태 처리가 필요하고, 때로는 렌더링 폭포(waterfall) 현상을 유발할 수 있습니다. `react-router-dom`의 `loader`는 이러한 문제를 해결하는 현대적인 데이터 페칭 방법입니다.

#### `loader` 함수란?

- **정의**: 라우트 설정에서 각 경로(`path`)에 연결하는 함수입니다.
- **실행 시점**: 해당 경로로 이동이 시작될 때, 즉 **컴포넌트가 렌더링되기 전**에 실행됩니다.
- **역할**: 페이지에 필요한 데이터를 미리 서버에서 가져오는(fetch) 역할을 합니다.
- **반환**: `Promise` 또는 실제 데이터를 반환할 수 있습니다. `fetch`의 `Response` 객체를 직접 반환하면, 라우터가 알아서 `response.json()`을 호출하고 `await` 처리해줍니다.

#### `loader` 파일 분리 및 구현

데이터 로딩 로직을 별도의 파일로 분리하여 관리하면 코드의 재사용성과 가독성이 향상됩니다.

```javascript
// src/loader/events-loader.js

// 이벤트 목록을 가져오는 로더
export const eventsListLoader = async () => {
    const response = await fetch('http://localhost:9000/api/events');

    // loader는 fetch 결과를 바로 리턴하는 경우 알아서 json을 추출한다.
    return response;
}

// 이벤트 상세 정보를 가져오는 로더
// loader 함수는 라우트의 동적 파라미터({params})를 인자로 받을 수 있다.
export const eventDetailLoader = async ({params}) => {
    return await fetch(`http://localhost:9000/api/events/${params.eventId}`);
}
```

#### `useLoaderData` 훅

- **역할**: 현재 라우트의 `loader` 함수가 반환한 데이터를 컴포넌트 내에서 쉽게 가져올 수 있게 해주는 훅입니다.
- **장점**: 데이터 로딩이 완료된 후에 컴포넌트가 렌더링되므로, 컴포넌트 내에서 별도의 로딩 상태(`useState`, `useEffect`)를 관리할 필요가 없습니다.

#### `loader`와 `useLoaderData` 사용 예시

**이벤트 목록 페이지 (`EventPage.jsx`)**

```jsx
// src/pages/EventPage.jsx
import React from 'react';
import { useLoaderData } from 'react-router-dom';
import EventList from '../components/EventList.jsx';

const EventPage = () => {

    // useLoaderData 훅을 호출하여 loader가 반환한 데이터를 가져온다.
    // 이 데이터는 컴포넌트 렌더링 시점에 이미 사용 가능하다.
    const eventList = useLoaderData();

    return (
        <EventList eventList={eventList}/>
    );
};

export default EventPage;
```

**이벤트 상세 페이지 (`EventDetailPage.jsx`)**

```jsx
// src/pages/EventDetailPage.jsx
import React from 'react';
import { useLoaderData } from 'react-router-dom';
import EventItem from '../components/EventItem.jsx';

const EventDetailPage = () => {
    
    // eventDetailLoader가 반환한 상세 이벤트 데이터를 가져온다.
    const detailEvent = useLoaderData();

    return (
        <EventItem event={detailEvent}/>
    );
};

export default EventDetailPage;
```

### 4\. 데이터 생성 및 삭제 (Fetch API 활용)

`loader`는 주로 GET 요청에 사용되며, 데이터 생성(POST), 수정(PUT/PATCH), 삭제(DELETE)는 컴포넌트 내의 이벤트 핸들러에서 `fetch`를 직접 사용하여 처리합니다.

#### 이벤트 생성 (`EventForm.jsx`)

`<form>` 제출 시 `fetch`를 사용하여 서버에 POST 요청을 보냅니다.

```jsx
// src/components/EventForm.jsx
import styles from './EventForm.module.scss';
import {useNavigate} from 'react-router-dom';

const EventForm = () => {
    const navigate = useNavigate();

    const handleSubmit = e => {
        e.preventDefault();
        const formData = new FormData(e.target);

        // 서버로 보낼 payload
        const payload = {
            title: formData.get('title'),
            desc: formData.get('description'),
            beginDate: formData.get('date'),
            imageUrl: formData.get('image')
        };
        
        // 서버에 POST 요청 (즉시 실행 함수 사용)
        (async () => {
            const response = await fetch('http://localhost:9000/api/events', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify(payload)
            });

            if (response.ok) {
                // 성공 시 목록 페이지로 리다이렉트
                navigate('/events');
            }
        })();
    };

    return (
        <form className={styles.form} noValidate onSubmit={handleSubmit}>
            {/* ... (form inputs) */}
        </form>
    );
};
export default EventForm;
```

#### 이벤트 삭제 (`EventItem.jsx`)

삭제 버튼 클릭 시 `fetch`를 사용하여 서버에 DELETE 요청을 보냅니다.

```jsx
// src/components/EventItem.jsx
import styles from './EventItem.module.scss';
import {useNavigate} from 'react-router-dom';

const EventItem = ({event}) => {
    const { id } = event;
    const navigate = useNavigate();

    const handleDelete = async () => {
        const confirmed = window.confirm('정말로 삭제하시겠습니까?');
        if (!confirmed) return;

        try {
            const response = await fetch(`http://localhost:9000/api/events/${id}`, {
                method: 'DELETE'
            });

            if (response.ok) {
                alert('삭제가 완료되었습니다!')
                navigate('/events');
            } else {
                alert('삭제에 실패했습니다.')
            }
        } catch(error) {
            console.error(error);
            alert('삭제 중 오류가 발생했습니다.')
        }
    };

    return (
        <article className={styles.event}>
            {/* ... (event details) */}
            <menu className={styles.actions}>
                <a href="#">Edit</a>
                <button onClick={handleDelete}>Delete</button>
            </menu>
        </article>
    );
};

export default EventItem;
```

-----

## Ⅳ. 오늘 배운 내용 요약

| 구분 | 내용 | 관련 파일 |
| :-- | :-- | :-- |
| **Spring Boot 복습** | RESTful API 서버 구축 (Controller, Service, Repository, DTO, Entity) | `EventController.java`, `EventService.java`, DTOs, `Event.java` |
| | CORS 전역 설정을 통한 다른 출처 요청 허용 | `CorsConfig.java` |
| **React 복습** | `react-router-dom`을 이용한 SPA 라우팅 및 중첩 레이아웃 구성 | `route-config.jsx`, `RootLayout.jsx`, `EventLayout.jsx` |
| **React 심화 (핵심)** | **`loader` 함수**: 컴포넌트 렌더링 전 데이터 페칭을 수행하여 성능과 사용자 경험 개선 | `events-loader.js`, `route-config.jsx` |
| | **`useLoaderData` 훅**: `loader`가 반환한 데이터를 컴포넌트에서 손쉽게 사용 | `EventPage.jsx`, `EventDetailPage.jsx` |
| | **`fetch` API**: 서버와 비동기 통신을 통해 데이터 CRUD(생성, 삭제) 기능 구현 | `EventForm.jsx`, `EventItem.jsx` |
