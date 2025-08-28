## 1. React Portal

**Portal**은 부모 컴포넌트의 DOM 계층 구조 바깥에 있는 다른 위치에 자식 컴포넌트를 렌더링하는 기능입니다. 주로 모달, 팝업, 툴팁 등 시각적으로 상위에 표시되어야 하지만, CSS `z-index`나 `overflow` 속성으로 인해 스타일 충돌이 발생할 수 있는 컴포넌트를 구현할 때 유용합니다.

### 핵심 개념

-   **DOM 분리**: 컴포넌트의 렌더링 위치를 현재 DOM 트리가 아닌 다른 곳으로 지정할 수 있습니다.
-   **이벤트 버블링**: Portal을 통해 렌더링된 컴포넌트의 이벤트는 React 컴포넌트 트리 구조를 따라 상위로 전파됩니다. DOM 트리상의 위치와 상관없이 React 트리상의 부모 컴포넌트에서 이벤트 처리가 가능합니다.

### 구현 방법

1.  **Portal을 렌더링할 DOM 노드 생성**
    `index.html` 파일에 Portal이 렌더링될 별도의 `<div>`를 추가합니다.

    ```html
    <!doctype html>
    <html lang="ko">
      <head>
        <meta charset="UTF-8" />
        <link rel="icon" type="image/svg+xml" href="/vite.svg" />
        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
        <title>리액트 기초 학습</title>
      </head>
      <body>

        <div id="backdrop-root"></div>
        <div id="modal-overlay-root"></div>

        <div id="root"></div>
        <script type="module" src="/src/main.jsx"></script>
      </body>
    </html>
    ```

2.  **Portal 컴포넌트 작성**
    `ReactDOM.createPortal()` 메서드를 사용하여 자식 컴포넌트를 지정된 DOM 노드에 렌더링합니다.

    -   `createPortal(child, container)`
        -   `child`: 렌더링할 React 자식 컴포넌트
        -   `container`: 자식을 렌더링할 실제 DOM 노드

    ```jsx
    // src/components/ui/Modal/Portal.jsx
    import React from 'react';
    import ReactDOM from 'react-dom';

    const Portal = ({ children, destId }) => {
        // 첫 번째 인자: 렌더링할 JSX
        // 두 번째 인자: 렌더링될 실제 DOM 요소
        return ReactDOM.createPortal(
            children,
            document.getElementById(destId)
        );
    };

    export default Portal;
    ```

3.  **Portal을 사용한 모달 컴포넌트**
    모달 컴포넌트에서 배경(Backdrop)과 모달 창(ModalOverlay)을 각각 다른 DOM 위치에 렌더링하기 위해 `Portal` 컴포넌트를 사용합니다.

    ```jsx
    // src/components/ui/Modal/ErrorModal.jsx
    import React from 'react';
    import Card from '../Card';
    import Button from '../Button';
    import styles from './ErrorModal.module.css';
    import Portal from './Portal.jsx';

    const BackDrop = ({onClose}) => {
        return <div className={styles.backdrop} onClick={onClose}></div>;
    };

    const ModalOverlay = ({title, message, onClose}) => {
        return (
            <Card className={styles.modal}>
                <header className={styles.header}><h2>{title}</h2></header>
                <div className={styles.content}><p>{message}</p></div>
                <footer className={styles.actions}>
                    <Button onClick={onClose}>Okay</Button>
                </footer>
            </Card>
        );
    }

    const ErrorModal = ({ title, message, onClose }) => {
        return (
            <>
               {/* 배경은 backdrop-root에 렌더링 */}
               <Portal destId='backdrop-root'>
                   <BackDrop onClose={onClose} />
               </Portal>

               {/* 모달창은 modal-overlay-root에 렌더링 */}
               <Portal destId='modal-overlay-root'>
                   <ModalOverlay title={title} message={message} onClose={onClose} />
               </Portal>
            </>
        );
    };

    export default ErrorModal;
    ```

---

## 2. `useRef` Hook

`useRef`는 렌더링 간에 값을 유지하고, 이 값의 변경이 리렌더링을 유발하지 않게 하는 Hook입니다. 주로 아래 두 가지 목적으로 사용됩니다.

1.  **DOM 요소에 직접 접근**: 특정 DOM 요소에 대한 참조를 생성하여 직접 조작할 때 사용합니다. (예: `focus()`, `value` 읽기)
2.  **리렌더링을 유발하지 않는 값 저장**: 컴포넌트의 생애주기 동안 유지되어야 하지만, 그 값이 바뀔 때마다 화면을 다시 그릴 필요는 없는 변수를 저장할 때 사용합니다. (예: 타이머 ID, 이전 상태 값)

### `useRef` vs `useState`

| 구분 | `useState` | `useRef` |
| :--- | :--- | :--- |
| **리렌더링** | Setter 함수 호출 시 컴포넌트 **리렌더링** | `.current` 값 변경 시 **리렌더링 안 됨** |
| **값 접근** | 상태 변수 이름으로 직접 접근 | `.current` 프로퍼티로 접근 |
| **주요 용도** | 화면에 표시될 상태 관리 | DOM 조작, 렌더링과 무관한 값 저장 |

### 사용 예시 1: DOM 요소 접근 (사용자 입력 처리)

`useState`를 사용하면 입력마다 리렌더링이 발생하지만, `useRef`는 제출 시점에만 값을 읽어오므로 불필요한 렌더링을 방지할 수 있습니다.

```jsx
// src/components/Users/AddUsers.jsx

import React, { useRef, useState } from 'react';
import styles from './AddUsers.module.css';
import Card from '../ui/Card';
import Button from '../ui/Button';
import ErrorModal from '../ui/Modal/ErrorModal.jsx';

const AddUsers = ({ onAddUser }) => {
    // useRef로 input 태그에 대한 참조 생성
    const usernameRef = useRef();
    const ageRef = useRef();

    const [error, setError] = useState(null);

    const handleSubmit = (e) => {
        e.preventDefault();

        // .current 프로퍼티를 통해 실제 input DOM 요소에 접근
        const enteredUsername = usernameRef.current.value;
        const enteredAge = ageRef.current.value;

        // ... 유효성 검사 로직 ...

        onAddUser({
            username: enteredUsername,
            age: enteredAge,
            id: Math.random().toString(),
        });

        // 입력 필드 초기화
        usernameRef.current.value = '';
        ageRef.current.value = '';
        usernameRef.current.focus();
    };

    return (
        <>
            {error && <ErrorModal title={error.title} message={error.message} onClose={() => setError(null)} />}
            <Card className={styles.input}>
                <form onSubmit={handleSubmit}>
                    <label htmlFor="username">이름</label>
                    <input id="username" type="text" ref={usernameRef} />
                    <label htmlFor="age">나이</label>
                    <input id="age" type="number" ref={ageRef} />
                    <Button type="submit">가입하기</Button>
                </form>
            </Card>
        </>
    );
};

export default AddUsers;
````

### 사용 예시 2: 리렌더링과 무관한 값 저장 (타이머 ID)

`setInterval`의 ID 값은 화면에 표시될 필요가 없으며, 리렌더링 시에도 값이 유지되어야 `clearInterval`을 정상적으로 호출할 수 있습니다. 이럴 때 `useRef`가 이상적입니다.

```jsx
// src/components/timerGame/TimerChallenge.jsx

import React, { useRef, useState } from 'react';
import ResultModal from './ResultModal.jsx';

const TimerChallenge = ({ title, targetTime }) => {
    // timerId는 리렌더링 되어도 값이 유지되어야 함
    const timerId = useRef();
    const dialogRef = useRef();
    const [timeRemaining, setTimeRemaining] = useState(targetTime * 1000);
    const timerIsActive = timeRemaining > 0 && timeRemaining < targetTime * 1000;

    if (timeRemaining <= 0) {
        clearInterval(timerId.current);
        dialogRef.current.showModal();
    }

    const handleStart = () => {
        // useRef의 값을 변경해도 리렌더링이 발생하지 않음
        timerId.current = setInterval(() => {
            setTimeRemaining(prevTime => prevTime - 10);
        }, 10);
    };

    const handleStop = () => {
        clearInterval(timerId.current);
        dialogRef.current.showModal();
    };

    // ... (생략) ...
};
```

-----

## 3\. Side Effect와 `useEffect` Hook

\*\*Side Effect(부수 효과)\*\*란 React 컴포넌트의 주된 목적인 'UI 렌더링' 외의 모든 작업을 의미합니다.

- HTTP 요청 (API 호출)
- 타이머 설정 (`setTimeout`, `setInterval`)
- DOM 직접 조작
- 로컬 스토리지 접근 등

`useEffect`는 이러한 Side Effect를 함수형 컴포넌트 내에서 수행할 수 있게 해주는 Hook입니다.

### `useEffect` 기본 사용법

```jsx
useEffect(() => {
  // Side Effect 로직 (콜백 함수)
  console.log('컴포넌트가 렌더링될 때마다 실행됩니다.');

  return () => {
    // Cleanup 함수 (정리 함수, 선택 사항)
    console.log('컴포넌트가 사라지거나, 다음 effect 실행 직전에 실행됩니다.');
  };
}, [/* 의존성 배열 */]);
```

### 의존성 배열(Dependency Array)

`useEffect`의 두 번째 인자로 전달하는 배열로, effect의 실행 시점을 제어합니다.

| 의존성 배열 | 실행 시점 |
| :--- | :--- |
| **생략** | 컴포넌트가 **렌더링될 때마다** 실행 |
| **`[]` (빈 배열)** | **최초 렌더링 시 한 번만** 실행 |
| **`[val1, val2]`** | 최초 렌더링 시 실행 + `val1` 또는 `val2`의 **값이 변경될 때마다** 실행 |

### 사용 예시: 로그인 폼 유효성 검사

사용자가 이메일이나 비밀번호를 입력할 때마다 유효성을 검사하면 불필요한 작업이 너무 많이 발생합니다. `useEffect`와 `setTimeout`을 사용하여 사용자의 입력이 멈춘 후 (예: 1초 후) 한 번만 유효성을 검사하도록 최적화할 수 있습니다.

```jsx
// src/components/SideEffect/Login.jsx

const Login = ({ onLogin }) => {
    const [enteredEmail, setEnteredEmail] = useState('');
    const [enteredPassword, setEnteredPassword] = useState('');
    const [formIsValid, setFormIsValid] = useState(false);

    // enteredEmail 또는 enteredPassword가 변경될 때마다 effect 실행
    useEffect(() => {
        // 1초 뒤에 유효성 검사를 실행하도록 타이머 설정
        const timerId = setTimeout(() => {
            console.log(`useEffect call in Login.js`);
            setFormIsValid(
                enteredEmail.includes('@') && enteredPassword.trim().length > 6
            );
        }, 1000);

        // Cleanup 함수:
        // 다음 effect가 실행되기 전, 또는 컴포넌트가 언마운트될 때 실행됨
        // 여기서는 이전 타이머를 취소하여 마지막 타이머만 실행되도록 보장
        return () => {
            console.log(`cleanup 실행!`);
            clearTimeout(timerId);
        };
    }, [enteredEmail, enteredPassword]); // 의존성 배열

    // ... (생략) ...
};
```

#### Cleanup 함수

`useEffect`의 콜백 함수에서 반환하는 함수로, "정리" 작업을 수행합니다.

- **실행 시점**:
    1.  컴포넌트가 화면에서 사라질 때 (언마운트)
    2.  의존성 배열의 값이 변경되어 다음 effect가 실행되기 직전
- **주요 용도**:
    - 설정했던 타이머 해제 (`clearTimeout`, `clearInterval`)
    - 추가했던 이벤트 리스너 제거 (`removeEventListener`)
    - API 구독 해지 등 메모리 누수를 방지하기 위한 작업

-----

## 4\. `forwardRef`

기본적으로 React 컴포넌트는 `ref` 속성을 props로 전달받을 수 없습니다. 부모 컴포넌트에서 자식 컴포넌트 내부의 특정 DOM 요소에 접근해야 할 때 `forwardRef`를 사용합니다.

`forwardRef`는 `ref`를 전달받아 자식 컴포넌트 내부의 DOM 요소에 연결해주는 역할을 합니다.

### 사용 예시: 모달 `dialog` 제어

부모 컴포넌트(`TimerChallenge`)에서 `useRef`로 생성한 `ref`를 자식 모달 컴포넌트(`ResultModal`)로 전달하여, 자식 내부의 `<dialog>` 요소의 `showModal()` 메서드를 직접 호출하는 예시입니다.

1.  **자식 컴포넌트에서 `forwardRef` 사용**
    컴포넌트를 `forwardRef`로 감싸고, 두 번째 인자로 `ref`를 받습니다. 이 `ref`를 내부의 DOM 요소에 연결합니다.

    ```jsx
    // src/components/timerGame/ResultModal.jsx
    import React, { forwardRef } from 'react';

    // 컴포넌트를 forwardRef로 감싸고, props와 ref를 인자로 받음
    const ResultModal = forwardRef(({ result, targetTime, ... }, ref) => {
        return (
            // 전달받은 ref를 dialog 요소에 연결
            <dialog ref={ref} className='result-modal'>
                <h2>Your {result}!</h2>
                {/* ... */}
                <form method='dialog'>
                    <button>Close</button>
                </form>
            </dialog>
        );
    });

    export default ResultModal;
    ```

2.  **부모 컴포넌트에서 `ref` 전달**
    `useRef`로 `ref` 객체를 생성하고, 자식 컴포넌트에 `ref` 속성으로 전달합니다.

    ```jsx
    // src/components/timerGame/TimerChallenge.jsx
    import React, { useRef, useState } from 'react';
    import ResultModal from './ResultModal.jsx';

    const TimerChallenge = ({ title, targetTime }) => {
        const timerId = useRef();
        // dialog 태그를 제어하기 위한 ref 생성
        const dialogRef = useRef();

        // ...

        const handleStop = () => {
            clearInterval(timerId.current);
            // 자식 컴포넌트의 dialog 요소에 직접 접근하여 메서드 호출
            dialogRef.current.showModal();
        };

        return (
            <>
                {/* 자식 컴포넌트에 ref를 props처럼 전달 */}
                <ResultModal
                    ref={dialogRef}
                    result="lost"
                    targetTime={targetTime}
                />
                {/* ... */}
            </>
        );
    };

    export default TimerChallenge;
    ```

<!-- end list -->
