# React 학습 노트

## 1\. React.js란?

### 1.1. 등장 배경

React.js는 Facebook의 소프트웨어 엔지니어인 Jordan Walke에 의해 개발되었습니다. 처음에는 Facebook의 뉴스 피드를 위해 개발되었고, 2013년에 오픈 소스로 공개되었습니다. 복잡한 사용자 인터페이스(UI)를 더 효과적으로 관리하고자 하는 필요성에서 탄생했습니다.

### 1.2. React.js vs Vanilla JS

기존의 Vanilla JS(순수 JavaScript)로 UI를 구현할 경우, UI의 복잡성과 동적인 변화를 관리하기 어려울 수 있습니다. React.js는 이러한 문제점들을 해결하기 위해 다음과 같은 이점을 제공합니다.

#### 1\) Component-Based Architecture (컴포넌트 기반 구조)

React는 UI를 **독립적이고 재사용 가능한 부분**으로 분할하여 관리합니다. 이를 \*\*컴포넌트(Component)\*\*라고 부릅니다. 이 구조는 코드의 재사용성을 높이고, 관리 및 유지 보수를 용이하게 만듭니다.

```jsx
// Welcome 이라는 컴포넌트는 재사용이 가능하며,
// 다른 프로퍼티(props)를 전달하여 커스터마이징할 수 있습니다.
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}

function App() {
  return (
    <div>
      <Welcome name="Alice" />
      <Welcome name="Bob" />
    </div>
  );
}

ReactDOM.render(
  <App />,
  document.getElementById('root')
);
```

#### 2\) Virtual DOM (가상 DOM)

React는 **Virtual DOM**을 사용하여 브라우저의 리렌더링 성능을 향상시킵니다. 데이터가 변경될 때마다 실제 DOM 전체를 새로 그리는 대신, 변경이 발생한 부분만 계산하여 실제 DOM에 업데이트합니다. 이 방식은 불필요한 렌더링을 최소화하여 성능을 크게 개선합니다.

#### 3\) Declarative Approach (선언형 접근 방식)

React는 선언형 프로그래밍 방식을 채택했습니다. 개발자는 UI가 '어떤 모습'이어야 하는지만 선언하면 되며, UI가 '어떻게' 업데이트될지에 대한 과정은 React가 알아서 처리합니다.

```jsx
// Timer 컴포넌트는 내부 상태(state)에 따라 자동으로 UI가 업데이트됩니다.
// 우리는 단지 상태가 어떻게 변하는지, 그에 따라 어떤 UI가 보여야 하는지만 정의하면 됩니다.
class Timer extends React.Component {
  constructor(props) {
    super(props);
    this.state = { seconds: 0 };
  }

  tick() {
    this.setState(state => ({
      seconds: state.seconds + 1
    }));
  }

  componentDidMount() {
    this.interval = setInterval(() => this.tick(), 1000);
  }

  componentWillUnmount() {
    clearInterval(this.interval);
  }

  render() {
    return (
      <div>
        Seconds: {this.state.seconds}
      </div>
    );
  }
}

ReactDOM.render(
  <Timer />,
  document.getElementById('root')
);
```

#### 4\) Strong Ecosystem and Community Support (강력한 생태계와 커뮤니티)

React는 거대한 커뮤니티와 강력한 생태계를 갖추고 있어, 다양한 라이브러리, 도구, 그리고 학습 자료를 쉽게 찾고 활용할 수 있습니다.

-----

## 2\. 프로젝트 설정 및 실행

### 2.1. 새로운 React 앱 만들기 (Vite)

1.  **Node.js 설치**: 먼저 컴퓨터에 Node.js가 설치되어 있어야 합니다.
2.  **Vite 프로젝트 생성**: 터미널에서 다음 명령어를 실행합니다.
    ```bash
    npm create vite@latest 프로젝트폴더명
    ```
    - 실행 중 `proceed(y)`를 입력하고, `React` -\> `Javascript`를 선택합니다.
3.  **프로젝트 폴더로 이동 및 의존성 설치**:
    ```bash
    cd 프로젝트폴더명
    npm install
    ```
4.  **개발 서버 실행**:
    ```bash
    npm run dev
    ```

### 2.2. GitHub에서 클론한 프로젝트 실행하기

1.  **Git Clone**:
    ```bash
    git clone <저장소_URL>
    ```
2.  **의존성 설치** (최초 한 번만 실행):
    ```bash
    npm install
    ```
3.  **개발 서버 실행** (서버를 켤 때마다 실행):
    ```bash
    npm run dev
    ```

<!-- end list -->

- 서버를 중지하려면 터미널에서 `Ctrl + C`를 누릅니다.

-----

## 3\. React Component

React는 **컴포넌트 기반** 라이브러리입니다. 컴포넌트는 재사용 가능한 코드 조각이며, 각각의 컴포넌트는 독립적인 기능을 수행합니다.

### 3.1. 컴포넌트의 종류

#### 1\) 함수 컴포넌트 (Functional Component) - **최신 방식**

가장 간단한 형태의 컴포넌트로, JavaScript 함수를 사용합니다. `props` 객체를 인자로 받아 React 엘리먼트(주로 JSX)를 반환합니다.

**`src/components/Bye.jsx`**

```jsx
// import React from 'react'; // 최신 React에서는 생략 가능
import Fruit from './Fruit.jsx';

// 함수형 컴포넌트: JavaScript 함수로 정의
function Bye () {
    return(
        // JSX에서는 반드시 하나의 부모 태그로 감싸야 합니다.
        // <React.Fragment> 또는 빈 태그 <> </>를 사용할 수 있습니다.
        <div>
            <a href="http://google.com">구글로 이동</a>
            {/* 다른 컴포넌트를 이렇게 태그처럼 가져와서 사용할 수 있습니다. */}
            <Fruit/>
        </div>
    );
}

export default Bye;
```

**`src/components/Fruit.jsx`**

```jsx
// rsc 단축키로 기본 구조 생성 가능
import React from 'react';

const Fruit = () => {
    return (
        <ul>
           <li>사과</li>
           <li>딸기</li>
           <li>오렌지</li>
        </ul>
    );
};

export default Fruit;
```

#### 2\) 클래스 컴포넌트 (Class Component) - **구형 방식**

ES6 클래스를 사용하여 정의합니다. `render()` 메서드가 반드시 있어야 하며, JSX를 반환합니다. 과거에는 상태(State) 관리와 라이프사이클 메서드 사용을 위해 클래스 컴포넌트를 사용했지만, 현재는 **Hooks**의 등장으로 함수 컴포넌트에서도 동일한 기능 수행이 가능해져 거의 사용되지 않습니다.

**`src/components/Hello.jsx`**

```jsx
import React from 'react';

// 클래스 컴포넌트: React.Component를 상속받는 클래스로 정의
class Hello extends React.Component {

    // render() 메서드에서 UI를 정의하고 반환
    render() {
        return (
            // <React.Fragment>의 단축 표현인 <>
            <>
                <h1>하하호호</h1>
                <h2>냠냠얌얌</h2>
            </>
        );
    }
}

export default Hello;
```

### 3.2. Props와 State

React 컴포넌트는 `props`와 `state`라는 두 가지 종류의 데이터를 다룹니다.

- **Props (Properties)**: **부모 컴포넌트**로부터 **자식 컴포넌트**에게 전달되는 데이터입니다. Props는 \*\*읽기 전용(Read-Only)\*\*이므로 자식 컴포넌트 내에서 직접 수정할 수 없습니다. (데이터의 단방향 흐름)
- **State**: 컴포넌트의 **내부 상태**를 관리하는 데이터입니다. State는 컴포넌트 내에서 변경이 가능하며, State가 변경되면 React는 해당 컴포넌트를 \*\*자동으로 다시 렌더링(re-rendering)\*\*하여 UI를 업데이트합니다.

-----

## 4\. JSX (JavaScript XML)

JSX는 JavaScript를 확장한 문법으로, React에서 UI를 표현하기 위해 사용됩니다. HTML과 유사하게 보이지만 실제로는 JavaScript 코드입니다. Babel과 같은 트랜스파일러에 의해 `React.createElement()` 함수 호출로 변환됩니다.

**`src/main.jsx` - React 앱의 시작점**

```jsx
import { createRoot } from 'react-dom/client'
import App from './App.jsx'

// 'root'라는 id를 가진 DOM 요소를 찾아 React 앱의 루트로 만듭니다.
// render 메서드를 통해 App 컴포넌트를 렌더링합니다.
createRoot(document.getElementById('root')).render(
    <App />
);
```

**JSX 주요 규칙:**

1.  **하나의 부모 요소**: 모든 엘리먼트는 반드시 하나의 부모 요소로 감싸져야 합니다.
2.  **JavaScript 표현식 사용**: 중괄호 `{}`를 사용하여 JSX 내부에 JavaScript 변수나 표현식을 포함할 수 있습니다.
3.  **HTML과 다른 속성 이름**: `class` 대신 `className`을, `for` 대신 `htmlFor`를 사용합니다. 이는 `class`와 `for`가 JavaScript의 예약어이기 때문입니다.
4.  **주석**: JSX 내에서는 `{/* 주석 내용 */}` 형식으로 주석을 작성합니다.

-----

## 5\. Props를 활용한 컴포넌트 제작

Props를 사용하면 컴포넌트에 동적인 데이터를 전달하여 재사용성을 높일 수 있습니다.

### 5.1. 데이터 전달 흐름

데이터는 부모 컴포넌트에서 자식 컴포넌트로, 마치 함수의 인자처럼 전달됩니다.

**`src/App.jsx` - 최상위 컴포넌트**

```jsx
import React from 'react';
import ExpenseList from './components/expenses/ExpenseList.jsx';
import NewExpense from './components/new-expense/NewExpense.jsx';
import CheckBoxStyle from './components/practice/CheckBoxStyle.jsx';

const App = () => {
    // 지출 항목에 대한 데이터 배열
    const expenseList = [
        { title: '닭강정', price: 8000, date: new Date(2025, 7, 13) },
        { title: '호두정과', price: 50000, date: new Date(2025, 8, 21) },
        { title: '리팩토링', price: 33000, date: new Date(2025, 4, 2) },
    ];

    return(
        <>
            <CheckBoxStyle/>
            <NewExpense/>
            {/* ExpenseList 컴포넌트에 'expenses'라는 이름의 prop으로 expenseList 배열을 전달 */}
            <ExpenseList expenses={expenseList}/>
        </>
    );
};

export default App;
```

**`src/components/expenses/ExpenseList.jsx` - 지출 목록**

```jsx
import React from 'react';
import './ExpenseList.css';
import ExpenseItem from './ExpenseItem.jsx';
import Card from '../ui/Card.jsx';

// App 컴포넌트로부터 받은 props 객체에서 expenses를 꺼내 사용
// ({expenses: expenseList})는 props.expenses를 expenseList라는 이름으로 사용하겠다는 의미(구조 분해 할당 + 별칭)
const ExpenseList = ({expenses: expenseList}) => {

    return (
        // Card 컴포넌트로 감싸서 공통 스타일을 적용
        <Card className='expenses'>
            {/* 전달받은 배열의 각 항목을 ExpenseItem 컴포넌트에 'expense' prop으로 전달 */}
            <ExpenseItem expense={expenseList[0]}/>
            <ExpenseItem expense={expenseList[1]}/>
            <ExpenseItem expense={expenseList[2]}/>
        </Card>
    );
};

export default ExpenseList;
```

> **참고**: 현재 코드는 배열의 각 항목을 수동으로 렌더링하고 있습니다. 실제 애플리케이션에서는 `map()` 함수를 사용하여 배열을 동적으로 렌더링하는 것이 일반적입니다.
>
> ```jsx
> {expenseList.map(expense => <ExpenseItem key={expense.id} expense={expense} />)}
> ```

**`src/components/expenses/ExpenseItem.jsx` - 개별 지출 항목**

```jsx
import React from 'react';
import './ExpenseItem.css';
import ExpenseDate from './ExpenseDate.jsx';

// ExpenseList로부터 받은 props에서 expense 객체를 사용
const ExpenseItem = ({expense}) => {
    // expense 객체에서 title, date, price를 추출
    const {title, date, price} = expense;

    // 가격을 원화 형식으로 변환
    const formatPrice = new Intl.NumberFormat('ko-KR').format(price);

    return (
        <div className="expense-item">
            {/* 날짜 관련 UI는 ExpenseDate 컴포넌트에 위임. date를 prop으로 전달 */}
            <ExpenseDate expenseDate={date} />
            <div className="expense-item__description">
                <h2>{title}</h2>
                <div className="expense-item__price">{formatPrice}원</div>
            </div>
        </div>
    );
};

export default ExpenseItem;
```

**`src/components/expenses/ExpenseDate.jsx` - 날짜 표시**

```jsx
import React from 'react';
import './ExpenseDate.css';

// ExpenseItem으로부터 받은 expenseDate prop을 사용
const ExpenseDate = ({expenseDate}) => {
    // toLocaleString을 사용하여 월 이름을 영어로 가져옴 (예: 'August')
    const month = expenseDate.toLocaleString('en-US',{month: 'long'}).slice(0,3);

    return (
        <div className='expense-date'>
            <div className='expense-date__month'>{month}</div>
            <div className='expense-date__year'>{expenseDate.getFullYear()}</div>
            <div className='expense-date__day'>{expenseDate.getDate()}</div>
        </div>
    );
};

export default ExpenseDate;
```

### 5.2. `props.children`을 이용한 컴포넌트 조합

`props.children`은 컴포넌트 태그 사이에 위치한 모든 내용을 담고 있습니다. 이를 이용해 다른 컴포넌트를 감싸는 **래퍼(Wrapper) 컴포넌트**를 만들 수 있습니다.

**`src/components/ui/Card.jsx` - 공통 카드 UI**

```jsx
import React from 'react';
import './Card.css';

// className의 기본값을 ''로 설정하여 오류 방지
const Card = ({children, className = ''}) => {
    // props.children은 이 컴포넌트가 감싸고 있는 모든 자식 요소를 의미합니다.
    // 외부에서 전달된 className을 추가로 적용할 수 있도록 함
    const finalClassName = `card ${className}`;
    
    return (
        <div className={finalClassName}>
            {children}
        </div>
    );
};

export default Card;
```

**`src/components/ui/Card.css`**

```css
.card {
    border-radius: 12px;
    box-shadow: 0 1px 10px rgba(0, 0, 0, 0.7);
}
```

위 `Card` 컴포넌트는 `ExpenseList`에서 다음과 같이 사용되어 내부에 있는 `ExpenseItem`들을 감싸고 공통적인 스타일(그림자, 둥근 모서리 등)을 적용합니다.

-----

## 6\. State와 이벤트 처리

`state`는 컴포넌트의 상태를 저장하고 변경하기 위한 데이터입니다. `state`가 변경되면 UI가 자동으로 다시 렌더링됩니다. 함수 컴포넌트에서는 `useState` Hook을 사용하여 `state`를 관리합니다.

### 6.1. `useState` Hook 사용법

`useState`는 React의 내장 함수(Hook)입니다.

- `useState(초기값)`을 호출하면 `[상태값, setter함수]` 형태의 배열을 반환합니다.
- **상태값(State)**: 현재 상태를 저장하는 변수.
- **setter 함수**: 상태값을 변경하는 함수. 이 함수를 통해서만 상태를 변경해야 React가 변화를 감지하고 UI를 업데이트합니다.

**`src/components/Counter.jsx` - 간단한 카운터 예제**

```jsx
import React, {useState} from 'react';

const Counter = () => {

    // count라는 이름의 state를 선언하고 초기값으로 10을 할당
    // setCount는 count 값을 변경하는 데 사용될 함수
    const [count, setCount] = useState(10);

    // 이벤트 핸들러: 버튼 클릭 시 호출될 함수
    const increaseHandler = () => setCount(count + 1);
    const decreaseHandler = () => setCount(count - 1);

    return (
        <div>
            <h1>숫자: {count}</h1>
            {/* 버튼 클릭(onClick) 시 해당 핸들러 함수를 연결 */}
            <button onClick={increaseHandler}>증가</button>
            <button onClick={decreaseHandler}>감소</button>
        </div>
    );
};

export default Counter;
```

### 6.2. 폼(Form)과 사용자 입력 처리

`useState`는 사용자의 입력을 실시간으로 관리하는 데 유용합니다.

**`src/components/new-expense/ExpenseForm.jsx`**

```jsx
import React, {useState} from 'react';
import './ExpenseForm.css';

const ExpenseForm = () => {

    // 여러 개의 입력 필드를 관리하기 위해 여러 개의 state를 선언
    const [title, setTitle] = useState('');
    const [price, setPrice] = useState(0);
    const [date, setDate] = useState(null);

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
        // form의 기본 동작(페이지 새로고침)을 막음
        e.preventDefault();

        // 수집된 state들을 하나의 객체로 만듦
        const payload = {title, price, date};
        console.log(payload);

        // 여기서 부모 컴포넌트로 데이터를 보내는 로직을 추가할 수 있음
    };

    return (
        <form onSubmit={handleSubmit}>
            <div className="new-expense__controls">
                <div className="new-expense__control">
                    <label>Title</label>
                    {/* onInput 이벤트 발생 시마다 setTitle 함수를 호출하여 title state를 업데이트 */}
                    <input type="text" onInput={e => setTitle(e.target.value)}/>
                </div>
                <div className="new-expense__control">
                    <label>Price</label>
                    <input
                        type="number"
                        min="100"
                        step="100"
                        // e.target.value는 문자열이므로 +를 붙여 숫자로 변환
                        onInput={e => setPrice(+e.target.value)}
                    />
                </div>
                <div className="new-expense__control">
                    <label>Date</label>
                    <input
                        type="date"
                        min="2019-01-01"
                        max={getTodayDate()}
                        onInput={e => setDate(e.target.value)}
                    />
                </div>
            </div>
            <div className="new-expense__actions">
                <button type="submit">Add Expense</button>
            </div>
        </form>
    );
};

export default ExpenseForm;
```

### 6.3. State를 이용한 동적 스타일링

State 변화에 따라 컴포넌트의 CSS 클래스를 변경하여 동적인 스타일을 적용할 수 있습니다.

**`src/components/practice/CheckBoxStyle.jsx`**

```jsx
import React, { useState } from 'react';
import './CheckBoxStyle.css';

const CheckBoxStyle = () => {

    // 체크박스의 체크 여부를 boolean 값으로 관리하는 state
    const [checked, setChecked] = useState(false);

    // 체크박스의 상태가 변경될 때마다 호출되는 핸들러
    const checkHandler = (e) => {
        // 이벤트 객체에서 현재 체크 상태를 가져와 state를 업데이트
        setChecked(e.target.checked);
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
                checked={checked}
            />
            {/* 삼항 연산자를 사용하여 'checked' state에 따라 클래스 이름을 동적으로 부여 */}
            <label className={checked ? 'checked' : 'unchecked'} htmlFor='styled-checkbox'>
                Check me!
            </label>
        </div>
    );
};

export default CheckBoxStyle;
```

**`src/components/practice/CheckBoxStyle.css`**

```css
.checkbox-container {
    display: flex;
    align-items: center;
    margin: 20px;
}

input[type="checkbox"] {
    display: none; /* 실제 체크박스는 숨김 */
}

label {
    cursor: pointer;
    padding: 10px 20px;
    border: 2px solid #ccc;
    border-radius: 5px;
    transition: background-color 0.3s, border-color 0.3s;
}

/* 체크되지 않았을 때의 스타일 */
label.unchecked {
    background-color: white;
    border-color: #ccc;
}

/* 체크되었을 때의 스타일 */
label.checked {
    background-color: #4caf50;
    border-color: #4caf50;
    color: white;
}
```