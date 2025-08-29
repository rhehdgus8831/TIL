## 1. 중첩 라우팅 (Nested Routing)

중첩 라우팅은 라우트 내에 또 다른 라우트를 정의하는 기능입니다. 이를 통해 특정 URL 경로 그룹(예: `/blog/*`)이 공통의 레이아웃을 공유하도록 만들 수 있습니다. 예를 들어, 블로그 목록 페이지와 블로그 상세 페이지는 모두 사이드바를 포함하는 동일한 레이아웃을 사용할 수 있습니다.

### 가. 중첩 레이아웃 컴포넌트 생성 (`BlogLayout.jsx`)

먼저 블로그 관련 페이지들만을 위한 `BlogLayout` 컴포넌트를 생성합니다. 이 레이아웃은 공통 UI(여기서는 `BlogSideBar`)를 포함하고, 자식 라우트 컴포넌트가 렌더링될 위치에 `Outlet`을 배치합니다.

```jsx
// src/layouts/BlogLayout.jsx
import { Outlet } from 'react-router-dom';
import styles from './BlogLayout.module.scss';
import BlogSideBar from '../components/BlogSideBar';

function BlogLayout() {
    return (
        <div className={styles.layout}>
            {/* 블로그 섹션의 공통 사이드바 */}
            <BlogSideBar />
            <main className={styles.main}>
                {/* 이 Outlet에는 BlogPage 또는 BlogPostDetailPage가 렌더링됩니다. */}
                <Outlet />
            </main>
        </div>
    );
}

export default BlogLayout;
````

### 나. 라우터 설정 수정 (`router-config.jsx`)

기존의 라우터 설정에서 `/blog` 경로 부분을 수정하여 중첩 구조를 만듭니다.

1.  `/blog` 경로의 `element`를 `BlogLayout`으로 지정합니다.
2.  기존의 블로그 페이지와 상세 페이지를 `BlogLayout`의 `children`으로 이동시킵니다.

<!-- end list -->

  - **상대 경로**: 자식 라우트의 `path`는 부모의 `path`에 상대적으로 결정됩니다. 예를 들어, `path: ':postId'`는 부모 경로인 `/blog`에 합쳐져 `/blog/:postId`가 됩니다.
  - **Index Route**: `index: true`로 설정된 라우트는 부모의 경로(`path: 'blog'`)와 정확히 일치할 때 `Outlet`에 렌더링될 기본 컴포넌트를 지정합니다.

<!-- end list -->

```jsx
// src/routes/router-config.jsx
import {createBrowserRouter} from 'react-router-dom';
import IndexPage from '../pages/IndexPage.jsx';
import BlogPage from '../pages/BlogPage.jsx';
import AboutPage from '../pages/AboutPage.jsx';
import RootLayout from '../layouts/RootLayout.jsx';
import ErrorPage from '../pages/ErrorPage.jsx';
import BlogPostDetailPage from '../pages/BlogPostDetailPage.jsx';
import BlogLayout from '../layouts/BlogLayout.jsx'; // BlogLayout 임포트

// 라우터 설정
export const router = createBrowserRouter([
    {
        path: '/',
        element: <RootLayout />, // 전체 페이지를 감싸는 최상위 레이아웃
        errorElement: <ErrorPage />,
        children: [
            {
                index: true,
                element: <IndexPage />
            },
            {
                path: 'blog',
                element: <BlogLayout />, // /blog 경로 그룹을 위한 중첩 레이아웃
                children: [
                    {
                        index: true, // /blog 경로 요청 시 BlogLayout의 Outlet에 렌더링
                        element: <BlogPage />
                    },
                    {
                        path: ':postId', // /blog/:postId 경로 요청 시 BlogLayout의 Outlet에 렌더링
                        element: <BlogPostDetailPage />
                    },
                ]
            },
            {
                path: 'about',
                element: <AboutPage />
            },
        ]
    }
]);
```

### 다. 사이드바 기능 구현 (`BlogSideBar.jsx`)

`BlogLayout` 내에서 사용될 `BlogSideBar`는 카테고리 목록, 최근 글 목록 등을 표시합니다. 여기서는 `useNavigate` 훅을 사용하여 특정 이벤트(버튼 클릭) 발생 시 페이지를 이동시키는 기능을 구현합니다.

| 훅 | 설명 |
| :--- | :--- |
| **`useNavigate`** | 페이지를 프로그래밍 방식으로 이동시키는 함수를 반환하는 훅입니다. `Link`나 `NavLink`를 사용할 수 없는 상황(예: 폼 제출 완료 후, 특정 로직 수행 후)에 유용합니다. |

```jsx
// src/components/BlogSideBar.jsx
import styles from './BlogSideBar.module.scss';
import {categories, posts} from '../dummy-data/dummy-post.js';
import {NavLink, useNavigate, useSearchParams} from 'react-router-dom';

const BlogSideBar = () => {
    const [searchParams, setSearchParams] = useSearchParams();
    // 새로고침 없이 페이지를 이동시키는 함수를 반환
    const navigate = useNavigate();

    const handleCategoryClick = (category) => {
        // 1. 먼저 /blog 경로로 페이지를 이동시킵니다.
        navigate(`/blog`);
        // 2. 그 다음, URL의 쿼리 스트링을 업데이트합니다.
        setSearchParams(prev => {
            prev.set('category', category);
            return prev;
        });
    };

    // ... (카테고리 카운트, 최근 글 목록 로직) ...

    return (
        <aside className={styles.sidebar}>
            <h2>카테고리</h2>
            <ul className={styles.categoryList}>
                {categories.map((category) => (
                    <li key={category.id}>
                        <button
                            className={`
                                ${styles.categoryButton}
                                ${
                                    (searchParams.get('category') || 'all') === category.id
                                        ? styles.active : ''
                                }
                            `}
                            onClick={() => handleCategoryClick(category.id)}
                        >
                            {category.name}
                            {/* ... */}
                        </button>
                    </li>
                ))}
            </ul>

            <div className={styles.recentPosts}>
                <h2>최근 글</h2>
                <ul>
                    {/* ... 최근 글 목록 렌더링 ... */}
                </ul>
            </div>
            {/* ... */}
        </aside>
    );
};

export default BlogSideBar;
```

-----

## 2\. 향상된 필터링 기능: 디바운싱(Debouncing) 적용

사용자가 검색창에 글자를 입력할 때마다 API를 요청하거나 상태를 변경하면 불필요한 리소스 낭비가 발생할 수 있습니다. **디바운싱**은 특정 시간 동안 추가적인 입력이 없으면 마지막에 한 번만 함수를 실행시키는 기법으로, 이러한 문제를 효과적으로 해결할 수 있습니다.

### `BlogFilter.jsx` 코드 분석

`useEffect`와 `setTimeout`을 사용하여 디바운싱을 구현합니다.

1.  `useState`를 사용해 입력창의 값을 실시간으로 관리합니다 (`searchValue`).
2.  `useEffect`를 사용하여 `searchValue`가 변경될 때마다 타이머를 설정합니다.
3.  사용자가 타이핑을 멈추고 500ms가 지나면, 타이머의 콜백 함수가 실행되어 `setSearchParams`로 URL 쿼리 스트링을 업데이트합니다.
4.  만약 500ms가 지나기 전에 새로운 입력이 들어오면, `useEffect`의 `cleanup` 함수(`return () => clearTimeout(timer)`)가 이전 타이머를 취소하고 새로운 타이머를 설정합니다.

<!-- end list -->

```jsx
// src/components/BlogFilter.jsx
import React, {useEffect, useState} from 'react';
import styles from './BlogFilter.module.scss';
import {categories} from '../dummy-data/dummy-post.js';
import {useSearchParams} from 'react-router-dom';

const BlogFilter = () => {
    const [searchParams, setSearchParams] = useSearchParams();
    // 입력창의 값을 관리하기 위한 상태
    const [searchValue, setSearchValue] = useState(searchParams.get('search') || '');

    // ... (handleCategoryChange, handleSortChange 생략) ...

    const handleSearch = e => {
        // 입력할 때마다 searchValue 상태를 즉시 업데이트
        setSearchValue(e.target.value);
    };

    // 디바운싱을 통한 검색어 처리
    useEffect(() => {
        // 500ms 후에 실행될 타이머 설정
        const timer = setTimeout(() => {
            setSearchParams(prev => {
                const trimmedSearch = searchValue.trim();
                if (trimmedSearch) {
                    // 검색어가 있으면 search 파라미터 설정
                    prev.set('search', trimmedSearch);
                } else {
                    // 검색어가 비어있으면 search 파라미터 제거
                    prev.delete('search');
                }
                return prev;
            });
        }, 500); // 500ms 지연

        // 컴포넌트가 리렌더링되거나 언마운트되기 전에 이전 타이머를 정리
        return () => clearTimeout(timer);
    }, [searchValue, setSearchParams]); // searchValue가 변경될 때마다 이 effect 실행

    return (
        <div className={styles.filter}>
            {/* ... (select 태그 생략) ... */}

            <input
                type='text'
                placeholder='검색어를 입력하세요'
                onChange={handleSearch}
                value={searchValue} // input의 value를 searchValue 상태와 동기화
            />
        </div>
    );
};

export default BlogFilter;
```
