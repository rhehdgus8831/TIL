# Event Fullstack Application 학습 노트

## 1\. `react-router-dom`을 활용한 데이터 CUD (생성, 수정, 삭제)

`react-router-dom` v6.4부터 도입된 `action` 함수와 `<Form>` 컴포넌트를 사용하면, `useEffect`나 별도의 상태 관리 없이도 서버 데이터의 생성(Create), 수정(Update), 삭제(Delete) 작업을 보다 선언적이고 간편하게 처리할 수 있습니다.

### 1.1. `action` 함수란?

- `loader`가 컴포넌트 렌더링 전 데이터를 **읽어오는(Read)** 역할이라면, `action`은 데이터를 **변경하는(Create, Update, Delete)** 작업을 담당합니다.
- 라우트(Route) 설정에서 특정 경로에 `action` 함수를 바인딩하면, 해당 경로로 `POST`, `PUT`, `PATCH`, `DELETE`와 같은 비-`GET` 요청이 발생했을 때 함수가 트리거됩니다.
- `action` 함수는 서버 요청을 처리하고, 성공 시 `redirect`를 통해 다른 페이지로 이동시키는 등의 작업을 수행할 수 있습니다.

### 1.2. `<Form>` 컴포넌트

- `react-router-dom`에서 제공하는 특수한 폼 컴포넌트입니다.
- HTML의 `<form>` 태그와 유사하지만, 페이지를 새로고침하지 않고 `action` 함수로 요청을 보냅니다.
- `method` 속성을 통해 `POST`, `PUT`, `DELETE` 등의 HTTP 메서드를 지정할 수 있습니다.

### 1.3. 라우트 설정 (`route-config.jsx`)

`createBrowserRouter`를 사용하여 라우트를 설정할 때, 각 경로에 `action` 함수를 직접 연결합니다.

- `/events/new` 경로의 `POST` 요청은 `manipulateAction` 함수를 트리거합니다.
- `/events/:eventId` 경로의 `DELETE` 요청은 `deleteAction` 함수를 트리거합니다.
- `/events/:eventId/edit` 경로의 `PUT` 요청 역시 `manipulateAction` 함수를 트리거합니다.

<!-- end list -->

```jsx
// frontend/src/routes/route-config.jsx

import {createBrowserRouter} from 'react-router-dom';
import ErrorPage from '../pages/ErrorPage.jsx';
import HomePage from '../pages/HomePage.jsx';
import EventPage from '../pages/EventPage.jsx';
import RootLayout from '../layouts/RootLayout.jsx';
import EventDetailPage from '../pages/EventDetailPage.jsx';
import EventLayout from '../layouts/EventLayout.jsx';
import { eventsListLoader, eventDetailLoader } from '../loader/events-loader.js';
import NewEventPage from '../pages/newEventPage.jsx';
// action 함수들을 import 합니다.
import {saveAction as manipulateAction, deleteAction } from '../loader/events-actions.js';
import EditPage from '../pages/EditPage.jsx';

const router = createBrowserRouter([
    {
        path: '/',
        element: <RootLayout/>,
        errorElement: <ErrorPage/>,
        children: [
            // ... (HomePage, EventLayout 등)
            {
                path: 'events',
                element: <EventLayout/>,
                children: [
                    // ... (EventPage)
                    {
                        path: 'new',
                        element: <NewEventPage/>,
                        // action 함수는 CUD를 트리거
                        action: manipulateAction
                    },
                    {
                        path: ':eventId',
                        element: <EventDetailPage/>,
                        loader : eventDetailLoader,
                        // :eventId 경로로 DELETE 요청 시 deleteAction 실행
                        action : deleteAction
                    },
                    {
                        path: ':eventId/edit',
                        element: <EditPage/>,
                        loader : eventDetailLoader,
                        // :eventId/edit 경로로 PUT 요청 시 manipulateAction 실행
                        action: manipulateAction
                    },
                ]
            },
        ]
    },
]);

export default router;
```

### 1.4. `action` 함수 구현 (`events-actions.js`)

`action` 함수는 `request`와 `params` 객체를 인자로 받습니다.

- `request`: `fetch` API의 `Request` 객체와 유사하며, `formData()` 메서드를 통해 폼 데이터를 추출할 수 있습니다.
- `params`: URL의 동적 파라미터(`:eventId` 등)를 담고 있습니다.

#### 이벤트 생성 및 수정 (`saveAction`)

- `request.method`를 확인하여 '생성(POST)'과 '수정(PUT)' 요청을 분기 처리합니다.
- `request.formData()`를 통해 `<Form>` 컴포넌트에서 제출된 데이터를 추출합니다.
- `fetch`를 사용해 백엔드 API로 데이터를 전송합니다.
- 요청이 성공하면 `redirect` 함수를 리턴하여 이벤트 목록 페이지(`/events`)로 이동시킵니다.

<!-- end list -->

```javascript
// frontend/src/loader/events-actions.js

// 이벤트를 등록하는 함수
import {redirect} from 'react-router-dom';

export const saveAction = async ({ request , params }) => {

    // form에 입력한 값 가져오기
    const formData = await request.formData();

    // 서버로 보낼 payload
    const payload = {
        title: formData.get('title'),
        desc: formData.get('description'),
        beginDate: formData.get('date'),
        imageUrl: formData.get('image')
    };

    let requestUrl = 'http://localhost:9000/api/events';
    // 수정 요청일 경우 URL에 eventId 추가
    if (request.method === 'PUT'){
        requestUrl += `/${params.eventId}`;
    }

    const response = await fetch(requestUrl, {
        method: request.method, // Form의 method 속성에 따라 'POST' 또는 'PUT'
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify(payload)
    });

    if (!response.ok) {
        throw new Error('이벤트 생성 또는 수정에 실패했습니다.');
    }

    // 목록페이지로 리다이렉트
    return redirect('/events');
};
```

#### 이벤트 삭제 (`deleteAction`)

- `params.eventId`를 사용하여 삭제할 이벤트의 ID를 가져옵니다.
- `DELETE` 메서드로 서버에 삭제 요청을 보냅니다.
- 성공 시 목록 페이지로 리다이렉트합니다.

<!-- end list -->

```javascript
// frontend/src/loader/events-actions.js

// 삭제처리 액션 함수
export const deleteAction = async ({ params }) => {
    if (!confirm('정말 삭제하시겠습니까?')) return;

    console.log(`삭제 액션 함수 실행 !`);

    const res = await fetch(`http://localhost:9000/api/events/${params.eventId}`, {
        method: 'DELETE'
    });

    if (!res.ok) {
        throw new Error('이벤트 삭제에 실패했습니다.');
    }

    return redirect('/events');
}
```

### 1.5. 폼 컴포넌트 구현 (`EventForm.jsx`)

- 이벤트 생성과 수정을 위한 공용 폼 컴포넌트입니다.
- `react-router-dom`의 `<Form>`을 사용하여 `action` 함수를 트리거합니다.
- `method` prop을 받아 `POST` 또는 `PUT`으로 설정합니다.
- `defaultValue`를 사용하여 수정 시 기존 데이터를 폼에 채워줍니다.

<!-- end list -->

```jsx
// frontend/src/components/EventForm.jsx

import styles from './EventForm.module.scss';
import {useNavigate, Form} from 'react-router-dom';

const EventForm = ({method, event = {}}) => {

    // 새로고침 없이 페이지 이동
    const navigate = useNavigate();

    const {title, desc, 'img-url': image, 'start-date': date} = event;

    const formatDate = (dateString) => {
        // ... 날짜 형식 변환 로직
    };

    // route 설정에 있는 action 함수를 트리거하려면 Form이라는 컴포넌트가 필요하다.
    // 필수 속성으로 method속성을 지정해야함.
    return (
        <Form
            method={method} // 'POST' 또는 'PUT'
            className={styles.form}
            noValidate
        >
            <p>
                <label htmlFor='title'>Title</label>
                <input
                    id='title'
                    type='text'
                    name='title'
                    required
                    defaultValue={event ? title : ''}
                />
            </p>
            {/* ... image, date, description input ... */}
            <div className={styles.actions}>
                <button type='button' onClick={() => navigate('..')}>Cancel</button>
                <button>{method === 'POST' ? 'save' : 'update'}</button>
            </div>
        </Form>
    );
};

export default EventForm;
```

### 1.6. `useSubmit`: `<Form>` 없이 `action` 트리거하기

때로는 `<Form>` 컴포넌트가 아닌, 일반 버튼 클릭 등으로 `action` 함수를 실행하고 싶을 때가 있습니다. 이럴 때 `useSubmit` 훅을 사용합니다.

- `EventItem.jsx`의 삭제 버튼은 `useSubmit`을 사용하여 `DELETE` 요청을 트리거합니다.
- `submit(data, options)` 형태로 호출합니다.
    - `data`: 전송할 데이터. 여기서는 `null`입니다.
    - `options`: `method`, `action` 등을 지정하는 객체.

<!-- end list -->

```jsx
// frontend/src/components/EventItem.jsx

import styles from './EventItem.module.scss';
import {Link, useNavigate, useSubmit} from 'react-router-dom';

const EventItem = ({ event }) => {
    // ...
    // Form 컴포넌트 없이 action함수를 작동시키는 법
    const submit = useSubmit();

    const handleRemove = e => {
        // Form없이 action 함수 트리거
        // 첫번째 인자는 form 데이터, 두번째 인자는 요청 정보(메서드, action 경로 등)
        submit(null, {method:`DELETE`})
    };

    return (
        <article className={styles.event}>
            {/* ... */}
            <menu className={styles.actions}>
                <Link to='edit'>Edit</Link>
                <button onClick={handleRemove}>Delete</button>
            </menu>
        </article>
    );
};

export default EventItem;
```

-----

## 2\. 무한 스크롤 구현

사용자가 페이지 하단에 도달했을 때 다음 페이지의 데이터를 동적으로 불러오는 무한 스크롤 기능을 구현합니다. 백엔드는 `Slice`를 사용하여 페이징 처리를 하고, 프론트엔드는 `IntersectionObserver` API를 사용해 스크롤 이벤트를 감지합니다.

### 2.1. 백엔드: QueryDSL과 `Slice`를 이용한 페이징

무한 스크롤에서는 전체 데이터 개수(`totalElements`)가 필요 없으므로, `Page` 대신 `Slice`를 사용하는 것이 더 효율적입니다. `Slice`는 다음 페이지의 존재 여부(`hasNext`)만 알려줍니다.

#### **QueryDSL 설정**

- `QueryDslConfig.java`: `JPAQueryFactory`를 빈으로 등록하여 QueryDSL을 사용할 수 있도록 설정합니다.
- `build.gradle`: QueryDSL 관련 의존성을 추가합니다.

| 어노테이션 및 클래스      | 설명                                                                                             |
| ------------------------- | ------------------------------------------------------------------------------------------------ |
| `@Configuration`          | 해당 클래스가 스프링의 설정 파일임을 나타냅니다.                                                 |
| `@PersistenceContext`     | 영속성 컨텍스트 `EntityManager`를 주입받습니다.                                                  |
| `EntityManager`           | JPA의 핵심 클래스로, 엔터티를 관리(조회, 저장, 수정, 삭제)합니다.                                |
| `JPAQueryFactory`         | QueryDSL을 사용하여 타입-세이프(type-safe)한 쿼리를 작성할 수 있게 해주는 클래스입니다.            |

```java
// backend/src/main/java/com/study/event/config/QueryDslConfig.java

package com.study.event.config;

import com.querydsl.jpa.impl.JPAQueryFactory;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class QueryDslConfig {

    @PersistenceContext // JPA 컨테이너가 관리하는 EntityManager를 주입
    private EntityManager em;

    @Bean
    public JPAQueryFactory factory() {
        return new JPAQueryFactory(em);
    }
}
```

#### **커스텀 리포지토리 구현 (`EventRepositoryCustom`, `EventRepositoryImpl`)**

- **`Slice`의 `hasNext` 판별 로직**: 요청한 페이지 크기(`pageSize`)보다 **1개 더 많은** 데이터를 조회합니다. 조회된 데이터 수가 `pageSize`보다 크면 다음 페이지가 있는 것(`hasNext = true`)으로 판단하고, 마지막 데이터(6번째)는 제거한 후 리스트를 반환합니다.

<!-- end list -->

```java
// backend/src/main/java/com/study/event/repository/EventRepositoryImpl.java

package com.study.event.repository;

import com.querydsl.jpa.impl.JPAQueryFactory;
import com.study.event.domain.entity.Event;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Slice;
import org.springframework.data.domain.SliceImpl;
import org.springframework.stereotype.Repository;

import java.util.List;
import static com.study.event.domain.entity.QEvent.*;

@Repository
@RequiredArgsConstructor
public class EventRepositoryImpl implements EventRepositoryCustom {

    private final JPAQueryFactory factory;

    @Override
    public Slice<Event> findEvents(Pageable pageable) {
        // 목록 조회
        List<Event> eventList = factory
                .selectFrom(event)
                .orderBy(event.createdAt.desc())
                .offset(pageable.getOffset())      // 몇 개를 건너뛸지
                .limit(pageable.getPageSize() + 1) // 요청한 사이즈보다 1개 더 조회
                .fetch();

        // 다음 페이지 존재 여부 확인
        boolean hasNext = false;
        if (eventList.size() > pageable.getPageSize()) {
            hasNext = true;
            // 실제로는 요청한 사이즈만큼만 리턴해야 하므로 마지막 데이터는 제거
            eventList.remove(eventList.size() - 1);
        }

        return new SliceImpl<>(eventList, pageable, hasNext);
    }
}
```

#### **서비스 및 컨트롤러**

- `EventService`: 리포지토리로부터 `Slice<Event>`를 받아 `hasNext`와 이벤트 목록(`eventList`)을 `Map`으로 감싸서 컨트롤러에 반환합니다.
- `EventController`: `page` 파라미터를 받아 서비스에 전달하고, 결과를 클라이언트에 JSON으로 응답합니다.

<!-- end list -->

```java
// backend/src/main/java/com/study/event/service/EventService.java

// ...
@Service
@RequiredArgsConstructor
@Transactional
public class EventService {
    private final EventRepository eventRepository;

    @Transactional(readOnly = true)
    public Map<String, Object> getEvents(int pageNo) {
        // Pageable 객체 생성 (페이지 번호는 0부터 시작하므로 -1)
        Slice<Event> eventSlice = eventRepository.findEvents(PageRequest.of(pageNo-1, 4));

        List<EventResponse> events = eventSlice.getContent()
                .stream()
                .map(EventResponse::from)
                .collect(Collectors.toList());

        return Map.of(
                "hasNext", eventSlice.hasNext(),
                "eventList", events
        );
    }
    // ...
}
```

### 2.2. 프론트엔드: `IntersectionObserver`를 사용한 무한 스크롤

`IntersectionObserver`는 타겟 요소가 뷰포트(viewport)와 교차하는지를 비동기적으로 감지하는 Web API입니다. 스크롤 이벤트를 직접 사용하는 것보다 성능상 이점이 있습니다.

#### **`EventPage.jsx` 구현**

1.  **상태(State) 관리**: `useState`를 사용하여 이벤트 목록(`eventList`), 현재 페이지 번호(`currentPage`), 데이터 종료 여부(`isFinish`), 로딩 상태(`loading`)를 관리합니다.
2.  **`IntersectionObserver` 생성**:
    - `useEffect` 내에서 `new IntersectionObserver()`를 통해 옵저버를 생성합니다.
    - 콜백 함수는 타겟 요소(`entries[0]`)가 화면에 나타나면(`isIntersecting`이 `true`이면) `fetchEvents` 함수를 호출합니다.
    - `threshold: 0.5` 옵션은 타겟 요소가 50% 보였을 때 콜백을 실행하도록 합니다.
3.  **감시 대상 설정**:
    - `useRef`로 `observerRef`를 생성하고, 페이지 하단의 보이지 않는 `<div>`에 연결합니다.
    - `observer.observe(observerRef.current)`를 통해 해당 `<div>` 감시를 시작합니다.
    - 컴포넌트가 언마운트될 때 `observer.disconnect()`로 감시를 해제하여 메모리 누수를 방지합니다.
4.  **데이터 Fetching (`fetchEvents`)**:
    - `loading` 상태를 `true`로 설정하여 중복 요청을 방지합니다.
    - 백엔드 API를 호출하여 데이터를 가져옵니다.
    - 가져온 데이터를 기존 `eventList`에 추가하고(`...prev`), `currentPage`를 1 증가시킵니다.
    - 서버에서 받은 `hasNext` 값으로 `isFinish` 상태를 업데이트합니다.
    - 데이터 로딩이 끝나면 `loading` 상태를 `false`로 변경합니다.

<!-- end list -->

```jsx
// frontend/src/pages/EventPage.jsx

import React, {useEffect, useRef, useState} from 'react';
import EventList from '../components/EventList.jsx';
import EventSkeleton from '../components/EventSkeleton.jsx';

const EventPage = () => {
    // 감시대상 태그에 대한 참조
    const observerRef = useRef();

    const [eventList, setEventList] = useState([]);
    const [currentPage, setCurrentPage] = useState(1);
    const [isFinish, setIsFinish] = useState(false); // 더 이상 가져올 데이터가 있는지 여부
    const [loading, setLoading] = useState(false); // 로딩 상태

    const fetchEvents = async () => {
        if (isFinish || loading) return; // 데이터가 끝났거나 로딩 중이면 return
        setLoading(true);

        // 강제로 1.5초의 로딩 부여 (UX 테스트용)
        await new Promise(r => setTimeout(r, 1500));

        const response = await fetch(`http://localhost:9000/api/events?page=${currentPage}`);
        const {hasNext, eventList: events} = await response.json();
        setEventList(prev => [...prev, ...events]);
        setCurrentPage(prev => prev + 1); // 페이지 번호 갱신
        setIsFinish(!hasNext); // 다음 페이지가 없으면 isFinish를 true로 설정
        setLoading(false);
    };

    useEffect(() => {
        // 무한스크롤을 위한 옵저버 생성
        const observer = new IntersectionObserver((entries) => {
            if (isFinish || loading) return;

            // isIntersecting: 감시 대상이 화면에 들어왔는지 여부
            if (entries[0].isIntersecting) {
                fetchEvents();
            }
        }, {
            // 관찰 대상의 높이가 50% 정도 보일 때 감지 실행
            threshold: 0.5
        });

        // 감시 대상 설정
        if (observerRef.current) {
            observer.observe(observerRef.current);
        }

        // 컴포넌트 언마운트 시 옵저버 연결 해제
        return () => observer.disconnect();
    }, [currentPage, loading, isFinish]); // currentPage가 변경될 때마다 useEffect 재실행

    return (
        <>
            <EventList eventList={eventList} />
            {/* 무한스크롤 옵저버를 위한 감시대상 태그 */}
            <div ref={observerRef} style={{ height: 100 }}>
                {/* 로딩 중일 때 스켈레톤 UI 표시 */}
                {loading && <EventSkeleton/>}
            </div>
        </>
    );
};

export default EventPage;
```