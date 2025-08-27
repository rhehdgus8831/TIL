## 1. 동적 스타일링

### 1-1. 동적 인라인 스타일 (inline-style)

컴포넌트의 상태(state)에 따라 스타일을 직접 변경해야 할 때 유용합니다. `style` prop에 JavaScript 객체를 전달하여 CSS 속성을 제어할 수 있습니다.

-   **핵심 개념**:
    -   JSX의 `style` 속성에는 문자열이 아닌 JavaScript 객체를 전달합니다.
    -   CSS 속성명은 카멜 케이스(camelCase)로 작성합니다 (예: `background-color` -> `backgroundColor`).
    -   상태 변수의 값에 따라 다른 스타일 값을 동적으로 할당할 수 있습니다.

-   **코드 예시 (`/src/components/CourseGoals/CourseInput.jsx`)**:
    -   사용자 입력 값이 유효하지 않을 때(`isValid`가 `false`) `label`과 `input`의 스타일을 동적으로 변경하는 코드입니다.

    ```jsx
    import React, {useState} from 'react';
    import styles from './CourseInput.module.css';
    import Button from '../ui/Button';

    const CourseInput = ({ onAdd }) => {
        // ... 생략 ...
        // 입력값에 오류가 있는지 여부를 상태관리
        const [isValid, setIsValid] = useState(null);

        // ... 생략 ...

        return (
            <form onSubmit={submitHandler}>
                {/* isValid 상태에 따라 동적으로 스타일을 적용했지만,
                  CSS Module을 사용하면서 아래 방식은 주석 처리되었습니다.
                */}
                <div className="...">
                    <label
                        // style={{ color: isValid !== false ? 'black' : 'red' }}
                    >나의 목표</label>
                    <input
                        type='text'
                        onInput={goalInputHandler}
                        value={enteredText}
                        // style={{
                        //   background: isValid !== false ? 'transparent' : 'salmon',
                        //   borderColor: isValid !== false ? 'black' : 'red'
                        // }}
                    />
                </div>
                <Button type='submit'>목표 추가하기</Button>
            </form>
        );
    };

    export default CourseInput;
    ```

    > **참고**: 인라인 스타일은 단일 요소에 간단한 동적 스타일을 적용할 때 편리하지만, 스타일 코드가 복잡해지고 재사용성이 떨어질 수 있습니다. 또한, `:hover` 같은 가상 클래스(pseudo-class)는 사용할 수 없습니다.

### 1-2. 동적 클래스 조작

상태에 따라 클래스 이름을 JSX 요소에 동적으로 추가하거나 제거하는 방식입니다. 일반 CSS 파일을 사용하거나 CSS Module과 함께 사용할 수 있습니다.

-   **핵심 개념**:
    -   JavaScript 템플릿 리터럴(백틱 `` ` ``)을 사용하여 `className`을 동적으로 구성합니다.
    -   삼항 연산자나 `if`문을 활용하여 조건에 따라 다른 클래스 이름을 할당합니다.

-   **실습 예제 1 (`/src/components/practice/CheckBoxStyle.jsx`)**:
    -   `checked` 상태에 따라 `label`의 클래스를 `checked` 또는 `unchecked`로 변경합니다.

    ```jsx
    // /src/components/practice/CheckBoxStyle.jsx
    import React, { useState } from 'react';
    import './CheckBoxStyle.css';

    const CheckBoxStyle = () => {
        const [checked, setChecked] = useState(false);

        const checkHandler = (e) => {
            setChecked(!checked);
        }

        /*
          1. input[checkbox]에 change이벤트가 걸려서
          2. check상태가 바뀔 때마다 상태변수를 논리값으로 업데이트하여
          3. label의 클래스를 유동적으로 변경해야함.
        */

        return (
            <div className='checkbox-container'>
                <input
                    type='checkbox'
                    id='styled-checkbox'
                    onChange={checkHandler}
                />
                <label 
                    className={checked ? `checked` : 'unchecked'} 
                    htmlFor='styled-checkbox'
                >
                    Check me!
                </label>
            </div>
        );
    };

    export default CheckBoxStyle;
    ```

    ```css
    /* /src/components/practice/CheckBoxStyle.css */
    label.unchecked {
        background-color: white;
        border-color: #ccc;
    }

    label.checked {
        background-color: #4caf50;
        border-color: #4caf50;
        color: white;
    }
    ```

-   **실습 예제 2 (`/src/components/CourseGoals/CourseInput.jsx`)**:
    -   `isValid` 상태가 `false`일 때 `invalid` 클래스를 추가하여 유효성 검사 실패 스타일을 적용합니다.

    ```jsx
    // /src/components/CourseGoals/CourseInput.jsx
    // ...
    const CourseInput = ({ onAdd }) => {
        // ...
        const { 'form-Control' : formControl , invalid } = styles;
        const [isValid, setIsValid] = useState(null);
        // ...

        return (
            <form onSubmit={submitHandler}>
                <div className={`${formControl} ${isValid === false ? invalid : ''}`}>
                    <label>나의 목표</label>
                    <input
                        type='text'
                        onInput={goalInputHandler}
                        value={enteredText}
                    />
                </div>
                <Button type='submit'>목표 추가하기</Button>
            </form>
        );
    };
    // ...
    ```

---

## 2. CSS Module

CSS 클래스 이름이 다른 컴포넌트의 클래스와 충돌하는 것을 방지해주는 기술입니다. Create React App(CRA)이나 Vite 같은 도구에 기본적으로 내장되어 있습니다.

-   **핵심 개념**:
    -   파일 이름을 `[컴포넌트명].module.css` 형식으로 작성합니다.
    -   컴포넌트에서 `import styles from './[컴포넌트명].module.css'` 와 같이 스타일 파일을 import 합니다.
    -   `styles` 객체에 클래스 이름이 프로퍼티로 저장됩니다.
    -   클래스 이름은 빌드 과정에서 고유한 해시값으로 변환되어 전역적인 이름 충돌을 막아줍니다.
    -   클래스 이름에 하이픈(-)이 포함된 경우, `styles['클래스-이름']`과 같이 대괄호 표기법으로 접근합니다.

-   **코드 예시 (`/src/components/CourseGoals/CourseInput.jsx`)**:

    ```jsx
    import React, {useState} from 'react';
    // CSS Module import
    import styles from './CourseInput.module.css';
    import Button from '../ui/Button';

    const CourseInput = ({ onAdd }) => {
        
        // console.log('styles: ', styles);
        // styles 객체 비구조화 할당. 하이픈이 있는 클래스명은 별칭을 사용
        const { 'form-control' : formControl , invalid } = styles;

        // ...

        const submitHandler = e => {
            e.preventDefault();
            // ...
        };
        // ...

        return (
            <form onSubmit={submitHandler}>
                {/* 템플릿 리터럴을 사용해 동적으로 클래스 적용 */}
                <div className={`${formControl} ${isValid === false ? invalid : ''}`}>
                    <label>나의 목표</label>
                    <input
                        type='text'
                        onInput={goalInputHandler}
                        value={enteredText}
                    />
                </div>
                <Button type='submit'>목표 추가하기</Button>
            </form>
        );
    };

    export default CourseInput;
    ```

    ```css
    /* /src/components/CourseGoals/CourseInput.module.css */
    .form-control {
        margin: 0.5rem 0;
    }

    .form-control label {
        font-weight: bold;
        display: block;
        margin-bottom: 0.5rem;
    }

    /* ... */

    .form-control.invalid input {
        border-color: red;
        background: salmon;
    }
    .form-control.invalid label {
        color: red;
    }
    ```

---

## 3. SCSS (Sass)

SCSS는 CSS의 확장 언어로, 변수, 중첩(Nesting), 믹스인(Mixin) 등의 기능을 사용하여 CSS를 더 효율적으로 작성할 수 있게 해줍니다.

-   **설치**:
    
    ```bash
    npm install sass
    ```
    
-   **핵심 개념**:
    -   **변수 (`$`)**: 색상, 폰트 크기 등 반복적으로 사용되는 값을 변수로 관리할 수 있습니다.
    -   **중첩 (Nesting)**: HTML 구조처럼 CSS 선택자를 중첩하여 작성할 수 있어 가독성이 향상됩니다.
    -   **믹스인 (`@mixin`, `@include`)**: 재사용 가능한 스타일 블록을 만들어 필요할 때마다 포함시킬 수 있습니다.

-   **코드 예시 (`/src/components/Todos` 관련 파일)**:
    -   SCSS Module (`.module.scss`)을 사용하여 컴포넌트 스코프를 유지하면서 SCSS 문법을 활용했습니다.

    ```scss
    /* /src/components/Todos/scss/TodoHeader.module.scss */
    // 변수 사용
    $my-dark-gray: #343a40;
    $w-basic: 100px;

    // header style (Nesting 적용)
    header {
      padding: 48px 32px 24px;
      border-bottom: 1px solid #bbb;

      h1 {
        font-size: 36px;
        color: $my-dark-gray;
      }
      .day {
        margin-top: 4px;
        color: #868e96;
        font-size: 21px;
      }
      .tasks-left {
        color: #20c997;
        font-size: 18px;
        margin-top: 40px;
        font-weight: 700;
      }
    }
    ```

    ```scss
    /* /src/components/Todos/scss/TodoItem.module.scss */
    // 조각 css (Mixin 정의)
    @mixin content-center {
      display: flex;
      justify-content: center;
      align-items: center;
    }

    .todo-list-item {
      display: flex;
      align-items: center;
      padding: 12px 0;
      
      .check-circle {
        // Mixin 사용
        @include content-center;
        /* ... */
      }

      /* ... */
    }


---

## 4. 실습: TodoList 만들기

오늘 배운 동적 스타일링, CSS Module, SCSS를 종합적으로 활용하여 TodoList 애플리케이션을 만들었습니다.

### 4-1. 컴포넌트 구조

-   **`TodoTemplate.jsx`**: 전체 레이아웃을 감싸는 최상위 컴포넌트. `todos` 상태를 관리하고, 자식들에게 상태와 상태 변경 함수를 props로 전달합니다.
-   **`TodoHeader.jsx`**: 오늘 날짜와 남은 할 일 개수를 표시합니다.
-   **`TodoInput.jsx`**: 새로운 할 일을 입력받는 폼과 추가 버튼을 담당합니다.
-   **`TodoMain.jsx`**: `todos` 배열을 순회하며 `TodoItem` 컴포넌트를 렌더링합니다.
-   **`TodoItem.jsx`**: 개별 할 일 항목을 표시하고, 완료 체크 및 삭제 기능을 담당합니다.

### 4-2. 핵심 로직 (`TodoTemplate.jsx`)

-   **상태 관리**: `useState`를 사용하여 `todos` 배열 상태를 관리합니다.
-   **데이터 추가 (`addHandler`)**: `TodoInput`으로부터 새로운 할 일 텍스트를 받아 `todos` 배열에 추가합니다.
-   **데이터 삭제 (`deleteHandler`)**: `TodoItem`으로부터 삭제할 항목의 `id`를 받아 `filter`를 사용해 해당 항목을 제외한 새 배열로 상태를 업데이트합니다.
-   **데이터 수정 (`checkHandler`)**: `TodoItem`으로부터 체크할 항목의 `id`를 받아 `map`을 사용해 해당 항목의 `boolean` 값을 토글(toggle)합니다.

```jsx
// /src/components/Todos/TodoTemplate.jsx
import React, {useState} from 'react';
import TodoHeader from './TodoHeader.jsx';
import styles from './scss/TodoTemplate.module.scss';
import TodoMain from './TodoMain';
import TodoInput from './TodoInput';

const TodoTemplate = () => {

    const [todoInput, setTodoInput] = useState([
        {
            id: Math.random().toString(),
            boolean: false,
            data: "메롱"
        }
    ]);

    // 할 일 추가 함수
    const addHandler = (todoText) => {
        setTodoInput(prev => [...prev,{
            id : Math.random().toString(),
            boolean : false,
            data : todoText
        }]);
    }

    // 할 일 삭제 함수
    const deleteHandler = (id) => {
        setTodoInput(todoInput.filter(todoItem => todoItem.id !== id))
    }

    // 할 일 체크 함수
    const checkHandler = (id) => {
        setTodoInput(todoInput.map(todoItem => 
            todoItem.id === id 
                ? {...todoItem, boolean : !todoItem.boolean} 
                : todoItem
        ))
    }

    return (
        <div className={styles.TodoTemplate}>
            <TodoHeader todo={todoInput}/>
            <TodoMain 
                todo={todoInput} 
                onDelete={deleteHandler} 
                onCheck={checkHandler}
            />
            <TodoInput onAdd={addHandler}/>
        </div>
    );
};

export default TodoTemplate;
```

### 4-3. 동적 스타일 적용 (`TodoItem.jsx`)

- `todo.boolean` 값에 따라 `check-circle`과 `text`에 `active`, `finish` 클래스를 동적으로 부여하여 완료 상태 스타일을 적용합니다.

<!-- end list -->

```jsx
// /src/components/Todos/TodoItem.jsx
import React from 'react';
import { MdDelete, MdDone } from 'react-icons/md';
import styles from './scss/TodoItem.module.scss';

const TodoItem = ({todo, onDelete, onCheck}) => {
    const { text, remove, 'todo-list-item': itemStyle, 'check-circle': checkCircle , active, finish } = styles;

    return (
        <li className={itemStyle} >
            <div 
                className={`${checkCircle} ${todo.boolean ? active  : ''}`}
                onClick={() => onCheck(todo.id)}
            >
                <MdDone />
            </div>
            <span className={`${text} ${todo.boolean ? finish : ''}`}>
                {todo.data}
            </span>
            <div className={remove} onClick={() => onDelete(todo.id)}>
                <MdDelete />
            </div>
        </li>
    );
};

export default TodoItem;
```

