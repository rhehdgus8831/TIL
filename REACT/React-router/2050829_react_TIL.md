## 1. React Router 기본 설정

React Router는 React 애플리케이션에서 페이지 라우팅을 관리하기 위한 라이브러리입니다. 사용자가 요청한 URL에 따라 적절한 컴포넌트를 렌더링하여 SPA(Single Page Application)를 구현할 수 있게 해줍니다.

### 가. 라우터 설치

먼저, 프로젝트에 `react-router-dom` 라이브러리를 설치해야 합니다.

```bash
npm install react-router-dom
````

`package.json` 파일에서 설치된 것을 확인할 수 있습니다.

```json
"dependencies": {
  "react": "^19.1.1",
  "react-dom": "^19.1.1",
  "react-router-dom": "^7.8.2",
  "sass": "^1.91.0"
},
```

### 나. 라우터 설정 (`router-config.jsx`)

`createBrowserRouter` 함수를 사용하여 라우터 설정을 정의합니다. 각 경로는 객체 형태로 path와 element를 가집니다.

  - **`createBrowserRouter`**: 브라우저의 History API를 사용하여 URL과 UI를 동기화하는 라우터입니다.

| 프로퍼티 | 설명 |
| :--- | :--- |
| **`path`** | URL 경로를 지정합니다. |
| **`element`** | 해당 경로에 렌더링할 컴포넌트를 지정합니다. |
| **`children`** | 중첩된 라우트를 정의할 때 사용됩니다. (레이아웃 섹션 참고) |
| **`errorElement`**| 경로에 에러 발생 시 렌더링할 컴포넌트를 지정합니다. |
| **`index`** | `true`로 설정 시, 부모 경로와 정확히 일치할 때 렌더링될 기본 자식 라우트를 지정합니다. |

```jsx
// src/routes/router-config.jsx
import {createBrowserRouter} from 'react-router-dom';
import IndexPage from '../pages/IndexPage.jsx';
import BlogPage from '../pages/BlogPage.jsx';
import AboutPage from '../pages/AboutPage.jsx';
import RootLayout from '../layouts/RootLayout.jsx';
import ErrorPage from '../pages/ErrorPage.jsx';
import BlogPostDetailPage from '../pages/BlogPostDetailPage.jsx';

// 라우터 설정
export const router = createBrowserRouter([
    {
        path: '/',
        element: <RootLayout />,
        // custom error page 설정
        errorElement: <ErrorPage />,
        // children -> Layout의 Outlet부분을 뭘로 바꿀지를 설정
        children: [
            {
                index: true,
                element: <IndexPage />
            },
            {
                path: 'blog',
                element: <BlogPage />
            },
            {
                path: 'blog/:postId', // URL 파라미터를 받는 동적 라우트
                element: <BlogPostDetailPage />
            },
            {
                path: 'about',
                element: <AboutPage />
            },
        ]
    }
]);
```

### 다. 라우터 적용 (`main.jsx` -\> `App.jsx`)

`main.jsx`에서 `App` 컴포넌트를 렌더링하고, `App.jsx`에서 `RouterProvider`를 사용하여 정의된 라우터 설정을 애플리케이션에 적용합니다.

  - **`RouterProvider`**: `createBrowserRouter`로 생성된 라우터 객체를 받아 애플리케이션 전체에 라우팅 컨텍스트를 제공합니다.

<!-- end list -->

```jsx
// src/main.jsx
import {createRoot} from 'react-dom/client';
import App from './App.jsx';

createRoot(document.getElementById('root')).render(
    <App />
);
```

```jsx
// src/App.jsx
import React from 'react';
import {RouterProvider} from 'react-router-dom';
import {router} from './routes/router-config.jsx';

const App = () => {
    return (
        // 생성된 라우터 설정을 props로 전달
        <RouterProvider router={router}/>
    );
};

export default App;
```

-----

## 2\. 페이지 이동 (Link, NavLink)

React Router는 `<a>` 태그 대신 `Link`와 `NavLink` 컴포넌트를 사용하여 페이지를 새로고침 없이 이동할 수 있게 해줍니다.

### 가. `Link`

가장 기본적인 페이지 이동 컴포넌트입니다. `to` prop에 이동할 경로를 지정합니다.

```jsx
// src/pages/IndexPage.jsx
import styles from './IndexPage.module.scss';
import {Link} from 'react-router-dom';

const IndexPage = () => {
    return (
        <>
            <div className={styles.home}>
                <h1 className={styles.title}>개발자의 기술 블로그</h1>
                <p className={styles.subtitle}>React와 관련된 기술들을 공유합니다.</p>

                {/* to prop을 사용하여 /blog 경로로 이동하는 링크 생성 */}
                <Link to={'/blog'} className={styles.button}>
                    블로그 보러가기
                </Link>
            </div>
        </>
    );
};

export default IndexPage;
```

### 나. `NavLink`

`Link`와 유사하지만, 현재 경로와 `to` prop의 경로가 일치할 때 특정 스타일이나 클래스를 적용할 수 있는 기능이 추가되었습니다. 주로 내비게이션 메뉴를 만들 때 사용됩니다.

  - **`className` prop에 함수 전달**: `isActive`와 `isPending` 값을 인자로 받아 동적으로 클래스 이름을 결정할 수 있습니다.

| 파라미터 | 설명 |
| :--- | :--- |
| **`isActive`** | `NavLink`의 `to` prop 경로가 현재 URL과 일치하면 `true`가 됩니다. |
| **`isPending`** | 다음 페이지로의 전환이 진행 중일 때 `true`가 됩니다. (코드 스플리팅 등) |

```jsx
// src/components/MainNav.jsx
import React from 'react';
import {NavLink} from 'react-router-dom';
import styles from './MainNav.module.scss';

const MainNav = () => {
    const {nav, navLink, active} = styles;

    // 현재 링크가 활성화되었을 때 'active' 클래스를 추가하는 함수
    const activate = ({isActive}) => {
        return `${navLink} ${isActive ? active : ''}`;
    };

    return (
        <nav className={nav}>
            <NavLink to="/" className={activate}>Home</NavLink>
            <NavLink to="/blog" className={activate}>Blog</NavLink>
            <NavLink to="/about" className={activate}>About</NavLink>
        </nav>
    );
};
export default MainNav;
```

-----

## 3\. 레이아웃 (Layout)

애플리케이션의 여러 페이지에서 공통적으로 사용되는 헤더, 푸터, 내비게이션 등을 묶어 **레이아웃 컴포넌트**로 만들 수 있습니다.

### 가. `RootLayout.jsx` 작성

공통 UI 구조를 정의하고, `Outlet` 컴포넌트를 사용하여 자식 라우트의 컴포넌트가 렌더링될 위치를 지정합니다.

  - **`Outlet`**: 부모 라우트 컴포넌트에서 자식 라우트 컴포넌트를 렌더링하기 위한 placeholder입니다.

<!-- end list -->

```jsx
// src/layouts/RootLayout.jsx
import React from 'react';
import styles from './RootLayout.module.scss';
import MainNav from '../components/MainNav.jsx';
import {Outlet} from 'react-router-dom'; // Outlet 임포트

const RootLayout = () => {
    return (
        <div className={styles.layout}>
            <header className={styles.header}>
                <div className={styles.container}>
                    <MainNav/>
                </div>
            </header>
            <main className={styles.main}>
                {/* 이 부분에 자식 컴포넌트들이 렌더링됩니다. */}
                <Outlet />
            </main>
            <footer className={styles.footer}>
                <div className={styles.container}>
                    <p>© 2025 개발자 블로그. All rights reserved.</p>
                </div>
            </footer>
        </div>
    );
};

export default RootLayout;
```

### 나. 라우터 설정에 레이아웃 적용

`router-config.jsx`에서 `RootLayout`을 최상위 `element`로 설정하고, 하위 페이지들을 `children` 배열에 정의하여 중첩 라우팅 구조를 만듭니다.

```jsx
// src/routes/router-config.jsx
export const router = createBrowserRouter([
    {
        path: '/',
        element: <RootLayout />, // RootLayout이 모든 하위 페이지를 감쌉니다.
        errorElement: <ErrorPage />,
        children: [ // Outlet에 렌더링될 자식 컴포넌트들
            { index: true, element: <IndexPage /> },
            { path: 'blog', element: <BlogPage /> },
            // ...
        ]
    }
]);
```

-----

## 4\. 에러 페이지 (Error Page)

정의되지 않은 경로로 접근하거나 라우팅 과정에서 에러가 발생했을 때 보여줄 커스텀 에러 페이지를 설정할 수 있습니다.

### 가. `ErrorPage.jsx` 작성

`useRouteError` 훅을 사용하여 발생한 에러 객체에 접근할 수 있습니다. 이를 통해 상태 코드(e.g., 404)나 에러 메시지에 따라 다른 내용을 보여줄 수 있습니다.

  - **`useRouteError`**: 라우팅 과정에서 발생한 에러 정보를 반환하는 훅입니다.

<!-- end list -->

```jsx
// src/pages/ErrorPage.jsx
import React from 'react';
import styles from './ErrorPage.module.scss';
import {Link, useRouteError} from 'react-router-dom';

const ErrorPage = () => {
    // 발생한 에러 정보 (throw로 던져지거나 404 에러 발생 시)를 가져오기
    const error = useRouteError();
    console.log(error);

    return (
        <div className={styles.error}>
            <h1 className={styles.title}>앗! 문제가 발생했어요</h1>
            <p className={styles.message}>
                {
                    // 404 에러인 경우와 그 외 에러를 구분하여 메시지 출력
                    error.status === 404
                    ? '페이지를 찾을 수 없습니다.' : error.message
                }
            </p>
            <Link to='/' className={styles.button}>
                홈으로 돌아가기
            </Link>
        </div>
    );
};

export default ErrorPage;
```

### 나. 라우터 설정에 에러 페이지 적용

`createBrowserRouter` 설정에서 `errorElement` 프로퍼티에 `ErrorPage` 컴포넌트를 지정합니다.

```jsx
// src/routes/router-config.jsx
export const router = createBrowserRouter([
    {
        path: '/',
        element: <RootLayout />,
        errorElement: <ErrorPage />, // 에러 발생 시 이 컴포넌트가 렌더링됩니다.
        children: [
            // ...
        ]
    }
]);
```

-----

## 5\. 동적 라우팅 (URL 파라미터와 쿼리 스트링)

정적인 경로 외에, URL의 일부를 변수로 사용하여 동적으로 페이지를 렌더링할 수 있습니다.

### 가. URL 파라미터 (`useParams`)

URL 경로의 일부를 변수처럼 사용할 때 사용합니다. 예를 들어 `/blog/1`, `/blog/2` 와 같이 각기 다른 게시글 상세 페이지를 보여줄 때 유용합니다.

1.  **라우터 설정**: 경로에 `:파라미터이름` 형식으로 동적 세그먼트를 추가합니다.

    ```jsx
    // src/routes/router-config.jsx
    // ...
    {
        path: 'blog/:postId', // 'postId'라는 이름의 URL 파라미터를 받음
        element: <BlogPostDetailPage />
    },
    // ...
    ```

2.  **컴포넌트에서 파라미터 읽기**: `useParams` 훅을 사용하면 라우터 설정에서 정의한 파라미터 이름(`postId`)을 키로 갖는 객체를 반환받을 수 있습니다.

    ```jsx
    // src/pages/BlogPostDetailPage.jsx
    import React from 'react';
    import {useParams} from 'react-router-dom';
    import {posts} from '../dummy-data/dummy-post.js';
    import styles from './BlogPostDetailPage.module.scss';

    const BlogPostDetailPage = () => {
        // useParams 훅을 호출하여 URL 파라미터 객체를 가져옴
        const {postId} = useParams(); // { postId: '1' } 형태

        // 더미데이터에서 해당 ID를 가진 게시글을 찾음
        // URL 파라미터는 문자열이므로, 숫자와 비교하기 위해 형변환(+)이 필요
        const foundPost = posts.find(post => post.id === +postId);

        // ...
        return (
            <article className={styles.post}>
                <h1>{foundPost.title}</h1>
                {/* ... */}
            </article>
        );
    };

    export default BlogPostDetailPage;
    ```

### 나. 쿼리 스트링 (`useSearchParams`)

URL의 `?` 뒤에 오는 `key=value` 쌍의 데이터를 다룰 때 사용합니다. 주로 필터링, 정렬, 검색 기능 구현에 사용됩니다.

  - **`useSearchParams`**: URL의 쿼리 스트링을 읽고 수정할 수 있는 기능을 제공하는 훅입니다. 배열을 반환합니다.
      - `[0]`: 현재 쿼리 스트링을 담고 있는 `URLSearchParams` 객체. `.get('key')`으로 값을 읽을 수 있습니다.
      - `[1]`: 쿼리 스트링을 업데이트하는 함수.

<!-- end list -->

1.  **쿼리 스트링 읽기 (`BlogPage.jsx`)**: `useSearchParams`를 호출하여 `searchParams` 객체를 얻고, `.get()` 메서드로 특정 키의 값을 읽습니다. 값이 없을 경우 `null`이 반환되므로, 기본값을 설정해주는 것이 좋습니다.

    ```jsx
    // src/pages/BlogPage.jsx
    import {useSearchParams} from 'react-router-dom';

    const BlogPage = () => {
        // useSearchParams는 배열을 리턴
        // 0번 인덱스: 쿼리스트링들을 모아놓은 객체
        // 1번 인덱스: 쿼리스트링을 생성할 수 있는 함수
        const [searchParams] = useSearchParams();

        // 'category' 쿼리 스트링 값을 읽고, 없으면 'all'을 기본값으로 사용
        const category = searchParams.get('category') || 'all';
        const search = searchParams.get('search') || '';
        const sort = searchParams.get('sort') || 'latest';
        // ...
    };
    ```

2.  **쿼리 스트링 생성 및 수정 (`BlogFilter.jsx`)**: `setSearchParams` 함수를 사용하여 쿼리 스트링을 변경합니다. 이 함수에 객체를 전달하거나, 이전 상태를 받아 수정하는 콜백 함수를 전달할 수 있습니다.

    ```jsx
    // src/components/BlogFilter.jsx
    import {useSearchParams} from 'react-router-dom';

    const BlogFilter = () => {
        // 쿼리스트링 생성/수정 기능 사용
        const [searchParams, setSearchParams] = useSearchParams();

        // 카테고리 select 변경 이벤트 핸들러
        const handleCategoryChange = e => {
            // setSearchParams를 사용하여 'category' 쿼리 스트링을 업데이트
            setSearchParams(prev => { // 이전 searchParams를 기반으로 수정
                prev.set('category', e.target.value);
                return prev; // 수정된 searchParams 객체를 반환
            });
        };
        // ...
        return (
            <div className={styles.filter}>
                {/* select의 value를 searchParams에서 읽어와 동기화 */}
                <select
                    onChange={handleCategoryChange}
                    value={searchParams.get('category') || 'all'}
                >
                  {/* ... */}
                </select>
                {/* ... */}
            </div>
        );
    };

    export default BlogFilter;
    ```

-----

## 6\. 필터링 및 정렬 기능 구현

`useSearchParams`를 활용하여 블로그 게시글 목록을 필터링하고 정렬하는 기능을 구현할 수 있습니다. `BlogPage.jsx`에서 `searchParams`로부터 `category`, `search`, `sort` 값을 읽어와 `filter`와 `sort` 메서드를 적용합니다.

```jsx
// src/pages/BlogPage.jsx
import React from 'react';
import styles from './BlogPage.module.scss';
import {posts} from '../dummy-data/dummy-post.js';
import PostCard from '../components/PostCard.jsx';
import {useSearchParams} from 'react-router-dom';
import BlogFilter from '../components/BlogFilter.jsx';

const BlogPage = () => {
    const [searchParams] = useSearchParams();

    const category = searchParams.get('category') || 'all';
    const search = searchParams.get('search') || '';
    const sort = searchParams.get('sort') || 'latest';

    return (
        <>
            <div className={styles.blog}>
                <BlogFilter/>
                <div className={styles.grid}>
                    {posts
                        // 1. 제목 또는 내용으로 검색
                        .filter(post =>
                            post.title.toLowerCase().includes(search.toLowerCase())
                            || post.excerpt.toLowerCase().includes(search.toLowerCase())
                        )
                        // 2. 카테고리로 필터링
                        .filter(post =>
                            category === 'all' || post.category === category
                        )
                        // 3. 정렬 (최신순, 오래된순)
                        .sort((a, b) =>
                            sort === 'latest'
                                ? new Date(b.date) - new Date(a.date) // 내림차순 (최신순)
                                : new Date(a.date) - new Date(b.date) // 오름차순 (오래된순)
                        )
                        .map(post => <PostCard key={post.id} post={post}/>)}
                </div>
            </div>
        </>
    );
};

export default BlogPage;
```

```
```
