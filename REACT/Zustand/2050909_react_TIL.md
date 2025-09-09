# Zustand

## 1\. Zustand 란?

**Zustand**는 React를 위한 작고 빠르며 확장 가능한 상태 관리 라이브러리입니다. Redux나 MobX와 같은 다른 상태 관리 라이브러리에 비해 매우 간단한 API를 가지고 있어 쉽게 배우고 사용할 수 있습니다. 별도의 Provider로 앱을 감싸줄 필요가 없어 코드 구조가 간결해집니다.

-----

## 2\. Zustand 설정 및 설치

Zustand를 프로젝트에 추가하려면 다음 명령어를 사용합니다.

```bash
npm install zustand
```

  * `package.json` 파일에서 `zustand`가 의존성에 추가된 것을 확인할 수 있습니다.

-----

## 3\. Store (저장소) 생성하기

Zustand의 `create` 함수를 사용하여 중앙 상태 저장소(Store)를 만듭니다. 이 저장소는 애플리케이션의 상태와 해당 상태를 변경하는 액션 함수들을 포함합니다.

### 3.1. 기본 카운터 Store

`counterStore.js`는 카운터의 상태(`count`, `showCounter`)와 이를 조작하는 함수들(`increment`, `decrement` 등)을 정의합니다.

  * `create(set => ({ ... }))`: `create` 함수는 콜백 함수를 인자로 받으며, 이 콜백은 `set` 함수를 파라미터로 가집니다.
  * `set` **함수**: 상태를 업데이트하는 데 사용됩니다. 이전 상태(state)를 받아 새로운 상태 객체를 반환하는 함수를 전달할 수 있습니다.

| `set` 함수 사용법                                     | 설명                                                                                                                                                             |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `set((state) => ({ count: state.count + 1 }))`        | 이전 상태(`state`)를 참조하여 새로운 상태를 계산하고 반환합니다. 이 방법은 이전 값에 의존하는 상태를 업데이트할 때 안전합니다.                                           |
| `set({ count: 0 })`                                   | 기존 상태와 관계없이 상태를 완전히 새로운 값으로 덮어씁니다.                                                                                                           |
| `set((state) => ({ showCounter: !state.showCounter }))` | `boolean` 값을 토글하는 예시입니다.                                                                                                                                 |

#### 📄 `src/zustand-practice/Store/counterStore.js`

```javascript
import {create} from 'zustand';

// 중앙 상태 저장소 생성
export const useCounterStore
    = create((set)=> ({

    // 전역관리할 상태값들을 배치
    count: 0,
    showCounter: true,

    // 상태값을 변경하는 액션함수들을 배치
    // set함수로 상태값을 변경할 수 있으며 콜백의 파라미터 state는 이전상태값 모음 객체를 의미
    increment: () => set((state) =>({ count: state.count + 1 })),
    decrement: () => set((state) =>({ count: state.count - 1})),
    multiply: (amount) => set((state) =>({ count: state.count * amount})),
    toggle: () => set((state) =>({ showCounter: !state.showCounter})),
}));
```

-----

## 4\. 컴포넌트에서 Store 사용하기

생성한 Store는 React 컴포넌트 내에서 Hook처럼 호출하여 상태와 액션 함수를 가져올 수 있습니다.

### 4.1. 카운터 컴포넌트

`ZustandCounter.jsx` 컴포넌트는 `useCounterStore`를 호출하여 `count`와 같은 상태 값과 `increment` 같은 액션 함수들을 직접 가져와 사용합니다. Store의 상태가 변경되면 이 상태를 구독하는 컴포넌트는 자동으로 리렌더링됩니다.

#### 📄 `src/zustand-practice/components/ZustandCounter.jsx`

```jsx
import styles from './ZustandCounter.module.scss';
import {useCounterStore} from '../Store/counterStore.js';

const ZustandCounter = () => {

    // 상태 구독: useCounterStore() 훅을 호출하면 스토어 객체가 반환됩니다.
    // 객체 디스트럭처링을 통해 필요한 상태와 함수를 추출합니다.
    const {count,showCounter, increment, decrement,multiply,toggle} = useCounterStore()
    //console.log('x',x);

    return (
        <main className={styles.counter}>
            <h1>Zustand Counter</h1>
            {/* showCounter 값에 따라 카운터 값을 조건부 렌더링 */}
            {showCounter && <div className={styles.value}>{count}</div>}
            <div
                style={{
                    display: 'flex',
                    gap: 8,
                    justifyContent: 'center',
                    marginBottom: 8,
                }}>
                {/* 각 버튼 클릭 시 스토어의 액션 함수를 호출 */}
                <button onClick={increment}>Increment</button>
                <button onClick={decrement}>Decrement</button>
                <button onClick={() =>multiply(3)}>×3</button>
            </div>
            <button onClick={toggle}>Toggle Counter</button>
        </main>
    );
};

export default ZustandCounter;
```

-----

## 5\. Middleware를 활용한 상태 유지

Zustand는 미들웨어(Middleware)를 지원하여 Store에 추가적인 기능을 쉽게 통합할 수 있습니다. `persist` 미들웨어는 상태를 Local Storage, Session Storage 등 다양한 저장소에 자동으로 저장하여 브라우저를 새로고침해도 상태가 유지되도록 합니다.

### 5.1. 인증 상태 Store (with `persist`)

`authStore.js`는 사용자의 로그인 상태(`isLoggedIn`)를 관리합니다. `persist` 미들웨어를 사용하여 이 상태를 Local Storage에 저장합니다.

  * `persist( (set) => ({...}), { ... } )`: 첫 번째 인자로 Store 설정 함수를, 두 번째 인자로 `persist` 옵션 객체를 받습니다.

| `persist` 옵션 | 설명                                                                                                                                       |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `name`         | Local Storage에 저장될 때 사용될 키(key) 이름입니다.                                                                                |
| `partialize`   | 저장소에 저장할 상태의 일부만 선택하는 함수입니다. 민감한 정보(예: 토큰)를 제외하고 특정 상태만 저장할 때 유용합니다. |

#### 📄 `src/zustand-practice/Store/authStore.js`

```javascript
import {create} from 'zustand';
import {persist} from 'zustand/middleware';

// 인증 관련 상태값 중앙 관리
export const useAuthStore = create(
    // persist 미들웨어를 사용하여 스토어의 상태를 영속적으로 관리합니다.
    persist((set) => ({

        // 관리할 상태값
        isLoggedIn: false,

        // 상태변경 함수 (액션 함수)
        login: () => set(() => ({isLoggedIn: true})),
        logout: () => set(() => ({isLoggedIn: false})),
    }),
        {
            name: 'auth', // localStorage에 저장될 Key 이름
            // partialize: 저장할 상태를 선택하는 옵션입니다.
            // 여기서는 전체 상태(state)에서 isLoggedIn 값만 localStorage에 저장하도록 설정했습니다.
            // 절대 민감정보 (ex - token) 로컬스토리지에 저장하면 안됨
            partialize: (state) => ({isLoggedIn: state.isLoggedIn})
        })
);
```

### 5.2. 인증 관련 컴포넌트

`Auth.jsx`는 로그인 폼을, `UserProfile.jsx`은 로그인 시 보여줄 프로필 화면을 나타냅니다. `Header.jsx`는 로그인 상태에 따라 다른 UI를 보여줍니다. 이 모든 컴포넌트들은 `useAuthStore`를 구독하여 `isLoggedIn` 상태와 `login`, `logout` 함수를 사용합니다.

#### 📄 `src/zustand-practice/components/Auth.jsx`

```jsx
import styles from './Auth.module.scss';
import {useAuthStore} from '../Store/authStore.js';

const Auth = () => {

    // useAuthStore로부터 login 함수를 가져옵니다.
    const {login} = useAuthStore()
    const handleSubmit = (e) => {
        e.preventDefault();
        // 폼 제출 시 login 함수를 호출하여 isLoggedIn 상태를 true로 변경합니다.
        login();
    };

    return (
        <main className={styles.auth}>
            <section>
                <form onSubmit={handleSubmit}>
                    <div className={styles.control}>
                        <label htmlFor='email'>Email</label>
                        <input
                            type='email'
                            id='email'
                        />
                    </div>
                    <div className={styles.control}>
                        <label htmlFor='password'>Password</label>
                        <input
                            type='password'
                            id='password'
                        />
                    </div>
                    <button>Login</button>
                </form>
            </section>
        </main>
    );
};

export default Auth;
```

#### 📄 `src/zustand-practice/components/Header.jsx`

```jsx
import styles from './Header.module.scss';
import {useAuthStore} from '../Store/authStore.js';

const Header = () => {

    // isLoggedIn 상태와 logout 함수를 스토어에서 가져옵니다.
    const {isLoggedIn,logout} = useAuthStore()

    return (
        <header className={styles.header}>
            <h1>Zustand Auth</h1>

            {/* isLoggedIn 상태가 true일 때만 네비게이션 메뉴를 보여줍니다. */}
            { isLoggedIn && <nav>
                <ul>
                    <li>
                        <a href="/public">My Products</a>
                    </li>
                    <li>
                        <a href="/public">My Sales</a>
                    </li>
                    <li>
                        {/* 로그아웃 버튼 클릭 시 logout 함수를 호출합니다. */}
                        <button onClick={logout}>Logout</button>
                    </li>
                </ul>
            </nav>}
        </header>
    );
};

export default Header;
```

#### 📄 `src/zustand-practice/components/UserProfile.jsx`

```jsx
import styles from './UserProfile.module.scss';

const UserProfile = () => {
    return (
        <main className={styles.profile}>
            <h2>My User Profile</h2>
        </main>
    );
};

export default UserProfile;
```

-----

## 6\. App 컴포넌트에서 Store 통합 관리

최상위 `App.jsx` 컴포넌트에서는 각 Store의 상태를 구독하여 조건부 렌더링을 수행합니다. 예를 들어, `useAuthStore`의 `isLoggedIn` 값에 따라 `UserProfile` 컴포넌트 또는 `Auth` 컴포넌트를 보여줍니다.

#### 📄 `src/App.jsx`

```jsx
import React, { useState } from 'react';

import './App.css';
import ZustandCounter from './zustand-practice/components/ZustandCounter.jsx';
// 각 스토어 훅을 임포트합니다.
import {useCounterStore} from './zustand-practice/Store/counterStore.js';
import Header from './zustand-practice/components/Header.jsx';
import Auth from './zustand-practice/components/Auth.jsx';
import {useAuthStore} from './zustand-practice/Store/authStore.js';
import UserProfile from './zustand-practice/components/UserProfile.jsx';

const App = () => {

    // 카운터 스토어의 상태를 구독 (현재 코드에서는 사용되지 않음)
    const {count} = useCounterStore();

    // 인증 스토어의 상태를 구독
    const {isLoggedIn} = useAuthStore();

    return (
        <>
            <Header/>
            {/* isLoggedIn 값에 따라 UserProfile 또는 Auth 컴포넌트를 렌더링 */}
            {isLoggedIn ? <UserProfile/> : <Auth/>}
            <ZustandCounter />
        </>
    );
};

export default App;
```


알겠습니다. 기존 Zustand 학습 노트에 `fetch`를 사용한 비동기 데이터 로딩 예제를 추가해 드리겠습니다.

-----

## 7\. 비동기 작업 및 데이터 Fetching

Zustand는 비동기 함수를 액션으로 정의하여 API 호출과 같은 작업을 손쉽게 처리할 수 있습니다. 로딩 및 에러 상태를 포함하여 비동기 데이터 흐름을 관리하는 예제는 다음과 같습니다.

### 7.1. Posts 데이터 관리 Store (with async action)

`postStore.js`는 외부 API로부터 게시물 목록을 가져오는 상태를 관리합니다.

  * **초기 상태**: `posts` 배열, API 호출 진행 상태를 나타내는 `loading`, 오류 발생 시 메시지를 저장할 `error`를 포함합니다.
  * **비동기 액션 (`fetchPosts`)**:
    1.  `set({ loading: true, error: null })`을 호출하여 로딩 상태를 활성화하고 이전 에러를 초기화합니다.
    2.  `try...catch` 구문을 사용하여 `fetch` API 호출을 실행합니다.
    3.  **성공 시**: `fetch`로 가져온 데이터를 `json()`으로 파싱하고, `set`을 호출하여 `posts` 상태를 업데이트하고 `loading`을 `false`로 설정합니다.
    4.  **실패 시**: `catch` 블록에서 에러 메시지를 `error` 상태에 저장하고 `loading`을 `false`로 설정합니다.

#### 📄 `src/zustand-practice/Store/postStore.js` (예시 파일)

```javascript
import { create } from 'zustand';

// 외부 API로부터 게시물 데이터를 가져와 관리하는 스토어
export const usePostStore = create((set) => ({
    // 관리할 상태값들
    posts: [],      // 게시물 목록
    loading: false, // 로딩 상태
    error: null,    // 에러 메시지

    // 비동기 액션 함수
    fetchPosts: async () => {
        // 1. API 호출 시작 직전, 로딩 상태를 true로 변경
        set({ loading: true, error: null });

        try {
            const response = await fetch('https://jsonplaceholder.typicode.com/posts');

            if (!response.ok) {
                throw new Error('데이터를 불러오는 데 실패했습니다.');
            }

            const data = await response.json();
            
            // 2. 데이터 로딩 성공 시, posts 상태를 업데이트하고 로딩 상태를 false로 변경
            set({ posts: data, loading: false });
        } catch (error) {
            // 3. 에러 발생 시, 에러 상태를 업데이트하고 로딩 상태를 false로 변경
            set({ error: error.message, loading: false });
        }
    },
}));
```

### 7.2. Posts 컴포넌트

`Posts.jsx` 컴포넌트는 `usePostStore`를 사용하여 게시물 데이터를 화면에 렌더링하고, 로딩 및 에러 상태에 따라 다른 UI를 보여줍니다.

  * `useEffect`를 사용하여 컴포넌트가 처음 마운트될 때 `fetchPosts` 액션을 호출합니다.
  * `loading` 상태가 `true`이면 "로딩 중..." 메시지를 표시합니다.
  * `error` 상태에 값이 있으면 에러 메시지를 표시합니다.
  * `posts` 배열을 `map` 함수로 순회하여 각 게시물의 제목을 목록으로 보여줍니다.

#### 📄 `src/zustand-practice/components/Posts.jsx` (예시 파일)

```jsx
import React, { useEffect } from 'react';
import { usePostStore } from '../Store/postStore.js'; // postStore 임포트

const Posts = () => {
    // post 스토어에서 상태와 액션을 가져옵니다.
    const { posts, loading, error, fetchPosts } = usePostStore();

    // 컴포넌트가 마운트될 때 fetchPosts 액션을 실행합니다.
    useEffect(() => {
        fetchPosts();
    }, [fetchPosts]); // fetchPosts는 스토어 생성 시 한 번만 생성되므로 의존성 배열에 추가해도 안전합니다.

    // 로딩 중일 때 보여줄 UI
    if (loading) {
        return <div>로딩 중...</div>;
    }

    // 에러 발생 시 보여줄 UI
    if (error) {
        return <div>에러: {error}</div>;
    }

    // 게시물 목록을 보여줄 UI
    return (
        <div>
            <h2>게시물 목록</h2>
            <button onClick={fetchPosts}>새로고침</button>
            <ul>
                {posts.map(post => (
                    <li key={post.id}>{post.title}</li>
                ))}
            </ul>
        </div>
    );
};

export default Posts;
```

이와 같이 Zustand의 액션 함수 내에서 `async/await`를 사용하여 비동기 로직을 간결하게 작성하고, `set` 함수를 통해 API 호출의 각 단계(시작, 성공, 실패)에 따른 상태를 업데이트하여 UI에 실시간으로 반영할 수 있습니다.
