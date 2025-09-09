# React Context API 학습 노트

## 1\. Context API란?

React Context API는 **상태 관리 라이브러리 없이** 컴포넌트 트리 전체에 걸쳐 전역적으로 데이터를 공유할 수 있게 해주는 기능입니다. 기존에는 부모 컴포넌트에서 자식 컴포넌트로 데이터를 전달하기 위해 **"Props Drilling"** (프로퍼티 내리꽂기) 방식을 사용해야 했습니다. 이는 컴포넌트 구조가 복잡해질수록 상태 관리를 어렵게 만드는 원인이 되었습니다.

Context API를 사용하면, 여러 컴포넌트에서 공통으로 사용하는 상태나 함수를 **중앙 저장소**에서 관리하고, 필요한 컴포넌트에서 직접 해당 데이터에 접근할 수 있어 코드의 가독성과 유지보수성이 향상됩니다.

이번 프로젝트에서는 장바구니 모달의 노출 상태, 장바구니에 담긴 상품 목록, 상품 추가/제거 함수 등을 Context로 관리했습니다.

-----

## 2\. Context 설정 및 Provider 생성

전역 상태 관리를 위한 첫 단계는 Context 객체를 생성하고, 해당 Context를 통해 상태를 제공(Provide)할 컴포넌트를 만드는 것입니다.

### 2.1. Context 객체 생성 (`cart-context.js`)

`createContext`를 사용하여 Context 객체를 생성합니다. 이 객체는 상태를 공유하는 컴포넌트 트리의 '저장소' 역할을 합니다. `defaultValue`를 설정하면, Provider로 감싸지지 않은 컴포넌트에서 Context를 사용하려 할 때 이 기본값이 사용됩니다. 또한, IDE의 자동완성 기능에도 도움을 줍니다.

```javascript
// src/context/cart-context.js

// 장바구니 앱에서 사용할 공유 상태값들을 중앙관리하는 저장소
import { createContext } from 'react';

// 자동완성을 위한 세팅
const defaultValue = {
    cartIsShown: false,
    openModal: () => {},
    closeModal: () => {},
    cartItems: [],
    addToCartItem: (cartItems) => {},
    removeToCartItem: (id) => {},
};

const CartContext = createContext(defaultValue);

export default CartContext;
```

### 2.2. Provider 컴포넌트 생성 (`CartProvider.jsx`)

Provider는 Context 객체를 통해 하위 컴포넌트들에게 공유할 상태와 함수를 `value` prop으로 전달하는 역할을 합니다.

  - **`useState`**: 장바구니 모달의 표시 여부(`cartIsShown`), 장바구니 상품 목록(`cartItems`)을 상태로 관리합니다.
  - **Handler Functions**: `handleShowCart`, `handleHideCart`, `handleAddToCartItem`, `handleRemoveToCartItem`과 같은 함수들을 정의하여 상태를 변경하는 로직을 캡슐화합니다.
  - **`CartContext.Provider`**: 이 컴포넌트로 `children`을 감싸고, `value` prop에 공유하고자 하는 상태와 함수들을 객체 형태로 전달합니다.

<!-- end list -->

```jsx
// src/context/CartProvider.jsx

import React, { useState } from 'react';
import CartContext from './cart-context.js';

const CartProvider = ({ children }) => {

    // 장바구니 배열을 상태 관리
    const [cartItems, setCartItems] = useState([]);

    // 장바구니 모달을 여닫는 상태변수
    const [cartIsShown, setCartIsShown] = useState(false);

    // 장바구니에 데이터를 추가하는 함수
    const handleAddToCartItem = (newItem) => {
        // 기존 장바구니에 이미 해당 아이템이 있는지 확인
        const existingCartItemIndex = cartItems.findIndex(cartItem => cartItem.id === newItem.id);
        const existingCartItem = cartItems[existingCartItemIndex];

        let updatedItems;

        if (existingCartItem) { // 기존에 있는 아이템이면 수량과 가격을 업데이트
            const updatedItem = {
                ...existingCartItem,
                amount: existingCartItem.amount + newItem.amount,
                price: existingCartItem.price + newItem.price
            };
            updatedItems = [...cartItems];
            updatedItems[existingCartItemIndex] = updatedItem;
        } else { // 새롭게 추가된 아이템이면 배열에 추가
            updatedItems = [...cartItems, newItem];
        }
        setCartItems(updatedItems);
    };


    // 장바구니 삭제 함수
    const handleRemoveToCartItem = (id) => {
        const existingCartItemIndex = cartItems.findIndex(item => item.id === id);
        const existingItem = cartItems[existingCartItemIndex];
        
        const updatedItems = [...cartItems];

        if (existingItem.amount === 1) { // 수량이 1이면 배열에서 완전히 제거
            updatedItems.splice(existingCartItemIndex, 1);
        } else { // 수량이 1보다 크면 수량과 가격을 1만큼 줄임
            const eachPrice = existingItem.price / existingItem.amount;
            const updatedItem = { ...existingItem, amount: existingItem.amount - 1, price: existingItem.price - eachPrice };
            updatedItems[existingCartItemIndex] = updatedItem;
        }
        setCartItems(updatedItems);
    };

    // 모달을 열어주는 함수
    const handleShowCart = () => setCartIsShown(true);

    // 모달을 닫아주는 함수
    const handleHideCart = () => setCartIsShown(false);

    // 컨텍스트가 실제로 관리할 중앙 상태값
    const initialValue = {
        cartIsShown,
        openModal: handleShowCart,
        closeModal: handleHideCart,
        cartItems,
        addToCartItem: handleAddToCartItem,
        removeToCartItem: handleRemoveToCartItem,
    };

    return (
        <CartContext.Provider value={initialValue}>
            {children}
        </CartContext.Provider>
    );
};

export default CartProvider;
```

-----

## 3\. Context 상태 소비(Consume)하기

`useContext` Hook을 사용하여 Provider가 제공하는 `value`에 접근할 수 있습니다.

### 3.1. `App.jsx`에서 Provider 적용

애플리케이션의 최상위 컴포넌트에서 `CartProvider`로 감싸주면, 모든 하위 컴포넌트가 Context의 데이터에 접근할 수 있게 됩니다.

```jsx
// src/App.jsx

import './App.css';
import CartProvider from './context/CartProvider.jsx';
import MainCart from './components/Layout/MainCart.jsx';

const App = () => {
    return (
        // value 속성에 하위 컴포넌트들이 공유할 상태값들을 명시
        <CartProvider>
            <MainCart />
        </CartProvider>
    );
};

export default App;
```

### 3.2. `useContext`를 사용한 상태 접근

각 컴포넌트에서는 `useContext(CartContext)`를 호출하여 Provider가 제공한 `value` 객체를 받아 필요한 상태나 함수를 사용할 수 있습니다.

**`HeaderCartButton.jsx` - 모달 열기 및 카트 뱃지 업데이트**

```jsx
// src/components/Layout/HeaderCartButton.jsx

import React, { useContext } from 'react';
import CartIcon from './CartIcon.jsx';
import styles from './HeaderCartButton.module.scss';
import CartContext from '../../context/cart-context.js';

const HeaderCartButton = () => {
    // useContext를 통해 CartContext의 openModal, cartItems를 가져옴
    const { openModal, cartItems } = useContext(CartContext);
    const { button, icon, badge } = styles;

    // 카트 배열에 있는 amount의 총합을 계산하여 뱃지에 표시
    const numberOfCart = cartItems.reduce((acc, curr) => acc + curr.amount, 0);

    return (
        <button className={button} onClick={openModal}>
            <span className={icon}>
                <CartIcon />
            </span>
            <span>My Cart</span>
            <span className={badge}>{numberOfCart}</span>
        </button>
    );
};

export default HeaderCartButton;
```

**`MainCart.jsx` - 모달 조건부 렌더링**

```jsx
// src/components/Layout/MainCart.jsx

import React, { useContext } from 'react';
import Cart from '../Cart/Cart.jsx';
import Header from './Header.jsx';
import Meals from '../Meals/Meals.jsx';
import CartContext from '../../context/cart-context.js';

const MainCart = () => {
    // cartIsShown 상태를 가져와 모달의 렌더링 여부를 결정
    const { cartIsShown } = useContext(CartContext);

    return (
        <>
            {cartIsShown && <Cart />}
            <Header />
            <div id="main">
                <Meals />
            </div>
        </>
    );
};

export default MainCart;
```

**`MealItem.jsx` - 장바구니에 아이템 추가**

`MealItemForm`에서 입력받은 수량(`amount`)을 `handleAddToCart` 함수를 통해 받아와, `addToCartItem` 함수로 Context에 정의된 장바구니 배열에 아이템을 추가합니다.

```jsx
// src/components/Meals/MealItem.jsx

import styles from './MealItem.module.scss';
import MealItemForm from './MealItemForm';
import { useContext } from 'react';
import CartContext from '../../context/cart-context.js';

const MealItem = ({ id, price, description, name }) => {
    // addToCartItem 함수를 Context에서 가져옴
    const { addToCartItem } = useContext(CartContext);
    const { meal, description: desc, price: priceStyle } = styles;
    const formatPrice = new Intl.NumberFormat('ko-KR').format(price);

    // MealItemForm으로부터 수량을 받아 장바구니에 추가하는 함수
    const handleAddToCart = (amount) => {
        const cartItem = {
            id,
            name,
            price: price * amount,
            amount
        };
        addToCartItem(cartItem);
    };

    return (
        <li className={meal}>
            <div>
                <h3>{name}</h3>
                <div className={desc}>{description}</div>
                <div className={priceStyle}>{formatPrice}원</div>
            </div>
            <div>
                <MealItemForm id={id} onAddToCart={handleAddToCart} />
            </div>
        </li>
    );
};

export default MealItem;
```

-----

## 4\. `useRef`를 이용한 데이터 처리

`MealItemForm` 컴포넌트에서는 `useRef`를 사용하여 `input` 요소에 직접 접근하고, '담기' 버튼 클릭 시 해당 `input`의 현재 값(수량)을 가져옵니다. 이 값은 props로 전달받은 `onAddToCart` 함수를 통해 부모 컴포넌트(`MealItem`)로 전달됩니다.

```jsx
// src/components/Meals/MealItemForm.jsx

import Input from '../UI/Input';
import styles from './MealItemForm.module.scss';
import { useRef } from 'react';

const MealItemForm = ({ id, onAddToCart }) => {
    // input 요소에 접근하기 위해 useRef 사용
    const inputRef = useRef();

    const handleSubmit = (e) => {
        e.preventDefault();
        // ref의 현재 값(수량)을 부모에게 받은 onAddToCart 함수로 전달
        onAddToCart(+inputRef.current.value);
    }

    return (
        <form className={styles.form} onSubmit={handleSubmit}>
            <Input
                ref={inputRef}
                label='수량'
                inputAttr={{
                    id: 'amount_' + id,
                    type: 'number',
                    min: '1',
                    max: '5',
                    step: '1',
                    defaultValue: '1',
                }}
            />
            <button>담기</button>
        </form>
    );
};

export default MealItemForm;
```

-----

## 5\. 장바구니 기능 구현

장바구니(`Cart.jsx`)와 장바구니 아이템(`CartItem.jsx`) 컴포넌트에서도 `useContext`를 사용하여 상태를 관리합니다.

### 5.1. `Cart.jsx` - 장바구니 목록 및 총액 표시

`cartItems` 배열을 받아와 목록을 렌더링하고, `reduce` 함수를 사용해 주문 총액을 계산합니다. `closeModal` 함수를 '닫기' 버튼에 연결합니다.

```jsx
// src/components/Cart/Cart.jsx

import React, { useContext } from 'react';
import styles from './Cart.module.scss';
import CartModal from './CartModal';
import CartContext from '../../context/cart-context.js';
import CartItem from './CartItem.jsx';

const Cart = () => {
    // closeModal과 cartItems를 Context에서 가져옴
    const { closeModal, cartItems } = useContext(CartContext);
    const { 'cart-items': cartItemStyle, total, actions, 'button--alt': btnAlt, button } = styles;

    // cartItems 배열의 총액을 계산
    const totalPrice = cartItems.reduce((acc, curr) => acc + curr.price, 0);

    return (
        <CartModal onClose={closeModal}>
            {/* 주문 내역 */}
            <ul className={cartItemStyle}>
                {cartItems.map((cartItem) => (
                    <CartItem key={cartItem.id} cart={cartItem} />
                ))}
            </ul>
            <div className={total}>
                <span>주문 총액</span>
                <span>{new Intl.NumberFormat('ko-KR').format(totalPrice)}원</span>
            </div>
            <div className={actions}>
                <button className={btnAlt} onClick={closeModal}>닫기</button>
                <button className={button}>주문</button>
            </div>
        </CartModal>
    );
};

export default Cart;
```

### 5.2. `CartItem.jsx` - 장바구니 내 아이템 수량 조절

장바구니 모달 내에서 각 아이템의 수량을 조절할 수 있도록 `+`, `-` 버튼에 각각 `addToCartItem`, `removeToCartItem` 함수를 연결합니다.

```jsx
// src/components/Cart/CartItem.jsx

import styles from './CartItem.module.scss';
import { useContext } from 'react';
import CartContext from '../../context/cart-context.js';

const CartItem = ({ cart }) => {
    // 수량 조절 함수들을 Context에서 가져옴
    const { addToCartItem, removeToCartItem } = useContext(CartContext);
    const { id, name, price, amount } = cart;
    const { 'cart-item': cartItem, summary, price: priceStyle, amount: amountStyle, actions } = styles;
    const formatPrice = new Intl.NumberFormat('ko-KR').format(price);

    // + 버튼 클릭 시 1개 수량만큼의 아이템 정보를 addToCartItem으로 전달
    const handleAddClick = e => {
        addToCartItem({
            id,
            name,
            amount: 1,
            price: price / amount // 1개의 가격을 계산하여 전달
        });
    }

    // - 버튼 클릭 시 아이템 id를 removeToCartItem으로 전달
    const handleRemoveClick = e => {
        removeToCartItem(id)
    }

    return (
        <li className={cartItem}>
            <div>
                <h2>{name}</h2>
                <div className={summary}>
                    <span className={priceStyle}>{formatPrice}</span>
                    <span className={amountStyle}>x {amount}</span>
                </div>
            </div>
            <div className={actions}>
                <button onClick={handleRemoveClick}>−</button>
                <button onClick={handleAddClick}>+</button>
            </div>
        </li>
    );
};

export default CartItem;
```

-----

## 6\. 핵심 정리

| 메서드/상태        | 역할                                                                 | 사용된 컴포넌트                                     |
| ------------------ | -------------------------------------------------------------------- | --------------------------------------------------- |
| `createContext`    | 전역 상태를 담을 Context 객체를 생성                                 | `cart-context.js`                                   |
| `Context.Provider` | 생성된 Context를 통해 하위 컴포넌트에 상태와 함수를 제공             | `CartProvider.jsx`, `App.jsx`                         |
| `useContext`       | Provider가 제공하는 값(상태, 함수)에 접근하여 사용                   | `MainCart`, `HeaderCartButton`, `MealItem`, `Cart`, `CartItem` |
| `cartIsShown`      | 장바구니 모달의 노출 여부를 결정하는 boolean 상태                    | `CartProvider`, `MainCart`                            |
| `cartItems`        | 장바구니에 담긴 상품들의 배열 상태                                   | `CartProvider`, `HeaderCartButton`, `Cart`, `CartItem` |
| `openModal`        | `cartIsShown` 상태를 `true`로 변경하여 모달을 여는 함수              | `CartProvider`, `HeaderCartButton`                    |
| `closeModal`       | `cartIsShown` 상태를 `false`로 변경하여 모달을 닫는 함수             | `CartProvider`, `Cart`                                |
| `addToCartItem`    | `cartItems` 배열에 새로운 상품을 추가하거나 기존 상품의 수량을 늘림  | `CartProvider`, `MealItem`, `CartItem`              |
| `removeToCartItem` | `cartItems` 배열에서 상품의 수량을 줄이거나 제거                     | `CartProvider`, `CartItem`                            |
| `useRef`           | DOM 요소(input)에 직접 접근하여 값을 읽어옴                          | `MealItemForm`                                      |
