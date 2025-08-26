
## 2-3. 상태(State)와 이벤트 처리

React의 핵심 개념인 \*\*상태(State)\*\*와 사용자 인터랙션을 처리하는 **이벤트**에 대해 학습합니다. 상태는 컴포넌트의 변경 가능한 데이터를 의미하며, 상태가 변경되면 컴포넌트가 다시 렌더링됩니다.

### 2-3-5. 입력창 초기화 (양방향 데이터 바인딩)

사용자 입력에 따라 상태를 업데이트하고, 상태가 변경되면 입력창의 값도 함께 변경되도록 하는 것을 **양방향 데이터 바인딩**이라고 합니다. `input` 태그의 `value` 속성에 상태 변수를 연결하고 `onChange` (또는 `onInput`) 이벤트 핸들러에서 상태를 업데이트하여 구현합니다.

#### 핵심 개념

- **상태 (State)**: 컴포넌트가 가지는 동적인 데이터. `useState` Hook으로 관리합니다.
- **이벤트 핸들러**: 사용자의 액션(클릭, 입력 등)에 반응하여 실행되는 함수.
- **양방향 바인딩**: UI 요소(e.g., input)의 값이 앱의 상태와 항상 동기화되는 것. 사용자가 값을 변경하면 상태가 업데이트되고, 상태가 코드에 의해 변경되면 UI 요소의 값도 업데이트됩니다.

#### `useState` Hook 사용법

`useState`는 React에서 상태 변수를 선언하고 관리하기 위한 Hook입니다.

| 반환값 | 설명 |
| :--- | :--- |
| `[state, setState]` | 배열의 첫 번째 요소는 현재 상태 값, 두 번째 요소는 해당 상태를 업데이트하는 setter 함수입니다. |
| `useState(initialState)` | `initialState`는 상태의 초기값을 설정합니다. |

#### 코드 예시: `ExpenseForm.jsx`

새로운 지출 항목을 추가하는 폼입니다. 제출 후 입력 필드를 초기화하기 위해 양방향 바인딩을 사용합니다.

```jsx
// src/components/new-expense/ExpenseForm.jsx

import React, {useState} from 'react';
import './ExpenseForm.css';

const ExpenseForm = ({ onAdd, onCancel }) => {

    const initUserInputState = {
        title: '',
        price: 0,
        date: null
    };

    // 여러 상태를 하나의 객체로 관리
    const [userInput, setUserInput] = useState(initUserInputState);

    // 오늘 날짜를 YYYY-MM-DD 형식으로 가져오는 함수
    const getTodayDate = () => {
        const today = new Date();
        const year = today.getFullYear();
        const month = String(today.getMonth() + 1).padStart(2, '0');
        const day = String(today.getDate()).padStart(2, '0');
        return `${year}-${month}-${day}`;
    };

    // form 제출 이벤트 핸들러
    const handleSubmit = e => {
        e.preventDefault(); // form의 기본 제출 동작(새로고침) 방지

        // 상위 컴포넌트로 제출할 데이터
        const expenseData = {
            ...userInput,
            date: new Date(userInput.date)
        };
        onAdd(expenseData); // 상위 컴포넌트로부터 받은 함수를 호출하여 데이터 전달

        // 제출 후 입력창 초기화
        setUserInput(initUserInputState);
    };

    // 제목 입력 이벤트 핸들러
    const titleChangeHandler = e => {
        setUserInput(prevUserInput => ({
            ...prevUserInput, // 기존 상태를 복사
            title: e.target.value, // title 프로퍼티만 변경
        }));
    };
    // ... (price, date 핸들러 생략)

    return (
        <form onSubmit={handleSubmit}>
            <div className="new-expense__controls">
                <div className="new-expense__control">
                    <label>Title</label>
                    <input
                        type="text"
                        onInput={titleChangeHandler}
                        value={userInput.title} // 상태 변수와 input의 value를 연결
                    />
                </div>
                <div className="new-expense__control">
                    <label>Price</label>
                    <input
                        type="number"
                        min="100"
                        step="100"
                        onInput={priceChangeHandler}
                        value={userInput.price || ''} // 상태 변수와 input의 value를 연결
                    />
                </div>
                <div className="new-expense__control">
                    <label>Date</label>
                    <input
                        type="date"
                        min="2019-01-01"
                        max={getTodayDate()}
                        onInput={dateChangeHandler}
                        value={userInput.date ?? ''} // 상태 변수와 input의 value를 연결
                    />
                </div>
            </div>
            <div className="new-expense__actions">
                <button type="button" className="cancel-btn" onClick={onCancel}>Cancel</button>
                <button type="submit">Add Expense</button>
            </div>
        </form>
    );
};

export default ExpenseForm;
```

-----

### 2-3-6. 객체로 상태값 관리하기

관련 있는 여러 상태값을 `useState`를 여러 번 사용하여 개별적으로 관리할 수도 있지만, 하나의 객체로 묶어서 관리하면 코드가 더 간결해질 수 있습니다.

#### 핵심 개념

- **상태 객체 업데이트**: 객체 상태를 업데이트할 때는 \*\*불변성(Immutability)\*\*을 유지하는 것이 중요합니다. 즉, 원본 객체를 직접 수정하는 대신, 스프레드 문법(`...`)을 사용하여 객체를 복사한 후 특정 프로퍼티만 수정하여 새로운 객체를 생성하고, 이 객체를 setter 함수에 전달해야 합니다.

#### 코드 예시: `ExpenseForm.jsx` (객체 상태 관리)

```jsx
// src/components/new-expense/ExpenseForm.jsx

// ...

// 초기 상태를 객체로 정의
const initUserInputState = {
    title: '',
    price: 0,
    date: null
};

// useState를 한 번만 사용하여 객체 전체를 상태로 관리
const [userInput, setUserInput] = useState(initUserInputState);

// 제목 입력 이벤트 핸들러
const titleChangeHandler = e => {
    // setter 함수에 함수를 전달하여 이전 상태값을 안전하게 사용
    setUserInput(prevUserInput => ({
        ...prevUserInput,       // 이전 상태 객체를 복사 (price, date는 유지)
        title: e.target.value, // title 프로퍼티만 새로운 값으로 덮어쓰기
    }));
};

const priceChangeHandler = e => setUserInput(prev => ({
    ...prev,
    price: +e.target.value,
}));

const dateChangeHandler = e => setUserInput(prev => ({
    ...prev,
    date: e.target.value,
}));

// ...
```

-----

### 2-3-7. 이전 상태값을 반영하여 상태 업데이트하기

상태 업데이트가 이전 상태값에 의존하는 경우, setter 함수에 값을 직접 전달하는 대신 **함수를 전달**하는 것이 더 안전하고 예측 가능합니다. React는 상태 업데이트를 비동기적으로 처리할 수 있기 때문입니다.

#### 핵심 개념

- **함수형 업데이트**: `setState` 함수에 콜백 함수를 전달하는 방식입니다. 이 콜백 함수는 인자로 \*\*최신 상태값(previous state)\*\*을 받아오며, 이 값을 기반으로 새로운 상태를 계산하여 반환합니다. React는 이 콜백 함수가 항상 최신 상태를 참조하도록 보장합니다.

#### 코드 예시: `Counter.jsx`

`increaseHandler`에서 `setCount`를 두 번 호출하는 예제입니다. 함수형 업데이트를 사용하지 않으면, 두 호출 모두 동일한 `count` 값을 참조하여 결국 1만 증가하게 됩니다. 함수형 업데이트를 사용하면 각 호출이 이전 업데이트가 적용된 최신 상태를 기반으로 동작하여 2가 증가합니다.

```jsx
// src/components/Counter.jsx

import React, {useState} from 'react';

const Counter = () => {

    const [count,setCount] = useState(10);

    const increaseHandler = () => {
        // 함수형 업데이트: 이전 상태(prev)를 기반으로 상태를 업데이트
        setCount((prev) => prev + 1);
        setCount((prev) => prev + 1); // 이 호출은 이전 호출이 끝난 후의 최신 상태를 참조
    }
    const decreaseHandler = () => setCount(count - 1);


    return (
        <div>
            <h1>숫자: {count}</h1>
            <button onClick={increaseHandler}>증가</button>
            <button onClick={decreaseHandler}>감소</button>
        </div>
    );
};

export default Counter;
```

-----

## 2-4. 컴포넌트 데이터 전달 (하향식 vs 상향식)

React의 데이터 흐름은 기본적으로 \*\*하향식(Top-Down)\*\*입니다. 부모 컴포넌트에서 자식 컴포넌트로 `props`를 통해 데이터를 전달합니다. 하지만 자식 컴포넌트에서 발생한 데이터를 부모 컴포넌트로 전달해야 할 경우(**상향식 데이터 전달**)가 있습니다.

### 2-4-1. 상향식 데이터 전달 (Lifting State Up)

자식 컴포넌트의 데이터를 부모 컴포넌트로 전달하기 위해, 부모 컴포넌트에서 **데이터를 처리할 함수**를 정의하고, 그 함수를 자식 컴포넌트에 `props`로 전달합니다. 자식 컴포넌트는 특정 이벤트가 발생했을 때 `props`로 받은 함수를 호출하며, 인자로 전달할 데이터를 넘겨줍니다.

#### 핵심 개념

- **상태 끌어올리기 (Lifting State Up)**: 여러 자식 컴포넌트가 공유해야 하는 상태가 있을 때, 그 상태를 가장 가까운 공통 조상 컴포넌트로 이동시키는 패턴입니다.

#### 코드 예시: `ExpenseFilter` -\> `ExpenseList`

`ExpenseFilter`(자식)에서 연도를 선택하면, 그 값을 `ExpenseList`(부모)로 전달하여 해당 연도의 지출 항목만 필터링합니다.

1.  **`ExpenseList.jsx` (부모 컴포넌트)**

    - `onFilterChange` 함수를 정의하고, 이 함수를 `ExpenseFilter`에 `onChangeFilter`라는 `props`로 전달합니다.
    - 선택된 연도를 관리하기 위한 `year` 상태를 가집니다.

    <!-- end list -->

    ```jsx
    // src/components/expenses/ExpenseList.jsx

    import React, {useState} from 'react';
    import './ExpenseList.css';
    import ExpenseFilter from './ExpenseFilter.jsx';
    // ...

    const ExpenseList = ({ expenses : expenseList }) => {
        // 선택된 연도를 상태로 관리
        const [year, setYear] = useState(new Date().getFullYear().toString());

        // 자식 컴포넌트(ExpenseFilter)로부터 데이터를 전달받을 함수
        const onFilterChange = (filteredYear) => {
            console.log(`선택된 연도: ${filteredYear}`);
            setYear(filteredYear); // 전달받은 연도로 상태 업데이트
        };

        // 선택된 연도와 일치하는 지출 항목만 필터링
        const filteredExpenses = expenseList
            .filter(ex => ex.date.getFullYear().toString() === year);

        return (
            <Card className='expenses'>
                {/* 자식에게 함수를 props로 전달 */}
                <ExpenseFilter onChangeFilter={onFilterChange} />
                {/* ... */}
            </Card>
        );
    };

    export default ExpenseList;
    ```

2.  **`ExpenseFilter.jsx` (자식 컴포넌트)**

    - `select` 태그의 `onChange` 이벤트가 발생하면 `props`로 받은 `onChangeFilter` 함수를 호출합니다.
    - 이때, 선택된 연도(`e.target.value`)를 인자로 전달합니다.

    <!-- end list -->

    ```jsx
    // src/components/expenses/ExpenseFilter.jsx

    import React from 'react';
    import './ExpenseFilter.css';

    // 부모로부터 onChangeFilter 함수를 props로 받음
    const ExpenseFilter = ({ onChangeFilter }) => {

        const changeYearHandler = e => {
            // props로 받은 함수를 호출하여 선택된 연도 값을 부모에게 전달
            onChangeFilter(e.target.value);
        };

        return (
            <div className='expenses-filter'>
                <div className='expenses-filter__control'>
                    <label>Filter by year</label>
                    <select onChange={changeYearHandler}>
                        <option value='2025'>2025</option>
                        {/* ... */}
                    </select>
                </div>
            </div>
        );
    };

    export default ExpenseFilter;
    ```

-----

## 3-1. 동적 리스트 렌더링

배열 데이터를 기반으로 동적으로 여러 개의 컴포넌트를 렌더링하는 방법입니다. JavaScript의 `map()` 배열 메서드를 주로 사용합니다.

### 3-1-1. `map()`을 이용한 렌더링

배열의 각 요소를 순회하며, 각 요소를 JSX 엘리먼트(컴포넌트)로 변환한 새로운 배열을 생성하여 렌더링합니다.

#### 코드 예시: `ExpenseList.jsx`

`filteredExpenses` 배열을 `map()`으로 순회하며 각 지출 항목에 대해 `<ExpenseItem>` 컴포넌트를 생성합니다.

```jsx
// src/components/expenses/ExpenseList.jsx
// ...
const ExpenseList = ({ expenses : expenseList }) => {
    // ...
    const filteredExpenses = expenseList
        .filter(ex => ex.date.getFullYear().toString() === year);

    return (
        <Card className='expenses'>
            {/* ... */}
            {
                // 배열을 map으로 순회하여 ExpenseItem 컴포넌트 배열을 생성
                filteredExpenses.map(ex => <ExpenseItem key={Math.random().toString()} expense={ex} />)
            }
        </Card>
    );
};
```

-----

### 3-1-2. `key` Props

리스트를 렌더링할 때 각 항목에 **고유하고 안정적인 `key`** prop을 제공해야 합니다. `key`는 React가 어떤 항목이 변경, 추가 또는 삭제되었는지 식별하는 데 도움을 줍니다. 이는 리렌더링 성능을 최적화하는 데 매우 중요합니다.

#### `key` Prop의 규칙

- **고유성**: `key`는 형제 엘리먼트 사이에서만 고유하면 됩니다.
- **안정성**: `key`는 렌더링 간에 변경되지 않아야 합니다. 항목의 순서가 바뀌더라도 동일한 항목은 동일한 `key`를 가져야 합니다.
- **`Math.random()` 이나 배열 인덱스는 피해야 합니다**: 리스트의 순서가 변경되거나 항목이 추가/삭제될 때 예기치 않은 동작을 유발할 수 있으므로, 데이터에 포함된 고유 ID를 사용하는 것이 가장 좋습니다.

#### 코드 예시: `CourseList.jsx`

`goals` 배열의 각 항목에는 고유한 `id` 프로퍼티가 있으므로, 이를 `key` 값으로 사용합니다.

```jsx
// src/components/CourseGoals/CourseList.jsx
import React from 'react';
import './CourseList.css';
import CourseItem from './CourseItem';

const CourseList = ({goals, onDelete }) => {

    return (
        <ul className='goal-list'>
            {
                goals.map(goal => (
                    <CourseItem
                        key={goal.id} // 데이터의 고유 id를 key로 사용
                        id={goal.id}
                        text={goal.text}
                        onDelete={onDelete}
                    />
                ))
            }
        </ul>
    );
};

export default CourseList;
```

-----

## 3-2. 조건부 렌더링

특정 조건에 따라 다른 컴포넌트나 JSX 엘리먼트를 렌더링하는 기법입니다.

### 3-2-1. 조건부 렌더링 기법

- **`if` 문 사용**: 렌더링 로직이 복잡할 때 변수나 함수를 사용하여 JSX 청크를 제어합니다.
- **삼항 연산자 (`? :`)**: 간단한 `if-else` 구문을 JSX 내에서 인라인으로 처리할 때 유용합니다.
- **논리 AND 연산자 (`&&`)**: 조건이 `true`일 때만 특정 엘리먼트를 렌더링하고 싶을 때 사용합니다.
- **즉시 실행 함수 표현식 (IIFE)**: 복잡한 로직을 JSX 내에서 바로 실행하고 싶을 때 사용할 수 있습니다.

#### 코드 예시: `ExpenseList.jsx` (삼항 연산자)

`filteredExpenses` 배열의 길이에 따라 지출 항목이 있을 때와 없을 때 다른 내용을 보여줍니다.

```jsx
// src/components/expenses/ExpenseList.jsx
// ...

const ExpenseList = ({ expenses : expenseList }) => {
    // ...
    const filteredExpenses = expenseList
        .filter(ex => ex.date.getFullYear().toString() === year);

    return (
        <Card className='expenses'>
            <ExpenseFilter onChangeFilter={onFilterChange} />
            <ExpenseChart expenses={filteredExpenses} />

            {
                // 삼항 연산자를 이용한 조건부 렌더링
                filteredExpenses.length > 0
                    // 조건이 true일 때: 지출 목록을 렌더링
                    ? filteredExpenses
                        .map(ex => <ExpenseItem key={Math.random().toString()} expense={ex} />)
                    // 조건이 false일 때: 메시지를 렌더링
                    : <p>아직 해당년도의 지출항목이 없습니다.</p>
            }
        </Card>
    );
};
```

#### 코드 예시: `NewExpense.jsx` (변수 사용)

`toggle` 상태에 따라 폼을 보여주거나 버튼을 보여줍니다.

```jsx
// src/components/new-expense/NewExpense.jsx
import React, {useState} from 'react';
import './NewExpense.css';
import ExpenseForm from './ExpenseForm';

const NewExpense = ({ onSave }) => {

    // 화면 상태를 제어하기 위한 논리상태 변수
    const [toggle, setToggle] = useState(false);

    // 조건에 따라 렌더링될 컴포넌트를 변수에 할당
    const formComponent = <ExpenseForm onAdd={onSave} onCancel={() => setToggle(false)}/>;
    const buttonComponent = <button onClick={() => setToggle(true)}>새로운 지출 추가하기</button>;

    return (
        <div className='new-expense'>
            {/* toggle 상태에 따라 다른 컴포넌트 렌더링 */}
            { toggle ? formComponent : buttonComponent }
        </div>
    );
};

export default NewExpense;
```

-----

## 실습: 목표 설정 앱 & 차트 UI

학습한 개념들을 종합하여 간단한 목표 설정 앱과 지출 내역 차트 UI를 만듭니다.

### 목표 설정 앱

- **`App.jsx`**: 최상위 컴포넌트로, 목표 목록(`goals`) 상태를 관리하고, 목표를 추가(`addGoalHandler`)하고 삭제(`deleteHandler`)하는 함수를 정의합니다.
- **`CourseInput.jsx`**: 목표를 입력하고 추가 버튼을 누르면 `onAdd` prop으로 받은 `addGoalHandler` 함수를 호출하여 입력값을 `App` 컴포넌트로 전달합니다.
- **`CourseList.jsx`**: `goals` 배열을 `props`로 받아 `map`을 이용해 여러 개의 `CourseItem`을 렌더링합니다.
- **`CourseItem.jsx`**: 각 목표 항목을 나타내며, 클릭 시 `onDelete` prop으로 받은 `deleteHandler` 함수를 호출하여 자신의 `id`를 전달합니다.

#### `App.jsx` (목표 설정 앱)

```jsx
// src/App.jsx

import React, {useState} from 'react';
import CourseList from './components/CourseGoals/CourseList.jsx';
import CourseInput from './components/CourseGoals/CourseInput.jsx';
import './App.css';

const App = () => {
    const goalList = [
        { id: 'g1', text: '테스트 데이터1' },
        { id: 'g2', text: '테스트 데이터2' },
    ];

    // 목표 데이터 배열을 상태로 관리
    const [goals, setGoals] = useState(goalList);

    // 새로운 목표를 추가하는 핸들러 (자식 CourseInput으로부터 호출됨)
    const addGoalHandler = (enteredGoal) => {
        setGoals(prev => {
            const newGoal = {
                id: `g${Math.random().toString()}`,
                text: enteredGoal,
            };
            return [...prev, newGoal]; // 기존 배열에 새 목표 추가
        });
    };

    // 목표를 삭제하는 핸들러 (자식 CourseItem으로부터 호출됨)
    const deleteHandler = (goalId) => {
        setGoals(prev => prev.filter(goal => goal.id !== goalId));
    };

    return (
        <div>
            <section id='goal-form'>
                <CourseInput onAdd={addGoalHandler} />
            </section>
            <section id='goals'>
                <CourseList goals={goals} onDelete={deleteHandler}/>
            </section>
        </div>
    );
};

export default App;
```

### 차트 UI

- **`ExpenseChart.jsx`**: 특정 연도의 `expenses` 데이터를 받아 월별로 지출액을 집계하고, `Chart` 컴포넌트에 필요한 데이터(`chartDataPoints`) 형식으로 가공하여 전달합니다.
- **`Chart.jsx`**: `dataPoints`를 받아 월별 막대그래프(`ChartBar`)를 렌더링하고, 연간 총 지출액을 계산하여 각 `ChartBar`에 `totalValue`로 전달합니다.
- **`ChartBar.jsx`**: `label`(월), `currentMonthValue`(월별 지출액), `totalValue`(연간 총액)을 받아 막대그래프의 채워지는 높이를 계산하고 시각적으로 표시합니다.

#### `ExpenseChart.jsx` (데이터 가공)

```jsx
// src/components/chart/ExpenseChart.jsx

import React from 'react';
import Chart from './Chart';

const ExpenseChart = ({ expenses }) => {

    const chartDataPoints = [
        { label: 'Jan', value: 0 }, { label: 'Feb', value: 0 },
        { label: 'Mar', value: 0 }, { label: 'Apr', value: 0 },
        { label: 'May', value: 0 }, { label: 'Jun', value: 0 },
        { label: 'Jul', value: 0 }, { label: 'Aug', value: 0 },
        { label: 'Sep', value: 0 }, { label: 'Oct', value: 0 },
        { label: 'Nov', value: 0 }, { label: 'Dec', value: 0 },
    ];

    // 전달받은 지출 배열을 통해 월별로 지출액을 합산
    expenses.forEach(ex => {
        const month = ex.date.getMonth(); // 0 = Jan, 1 = Feb, ...
        chartDataPoints[month].value += ex.price;
    });

    return <Chart dataPoints={chartDataPoints} />;
};

export default ExpenseChart;
```

#### `Chart.jsx` (차트 컨테이너)

```jsx
// src/components/chart/Chart.jsx

import React from 'react';
import ChartBar from './ChartBar';
import './Chart.css';

const Chart = ({ dataPoints }) => {
    // 연간 총 지출액 계산
    const totalValue = dataPoints.reduce((acc, dp) => acc + dp.value, 0);

    return (
        <div className='chart'>
            {
                dataPoints.map(dp =>
                    <ChartBar
                        key={dp.label}
                        label={dp.label}
                        currentMonthValue={dp.value}
                        totalValue={totalValue} // 각 막대그래프에 총액 전달
                    />)
            }
        </div>
    );
};

export default Chart;
```

#### `ChartBar.jsx` (개별 막대그래프)

```jsx
// src/components/chart/ChartBar.jsx

import React from 'react';
import './ChartBar.css';

const ChartBar = ({ label, currentMonthValue, totalValue }) => {
    let barFillHeight = '0%';

    if (totalValue > 0) {
        // 해당 월의 지출액 비율을 계산
        const percentage = (currentMonthValue / totalValue) * 100;
        barFillHeight = percentage + '%';
    }

    return (
        <div className='chart-bar'>
            <div className='chart-bar__inner'>
                {/* style 속성에 계산된 높이 값을 동적으로 적용 */}
                <div className='chart-bar__fill' style={{ height: barFillHeight }}></div>
            </div>
            <div className='chart-bar__label'>{label}</div>
        </div>
    );
};

export default ChartBar;
```