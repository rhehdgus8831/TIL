
# **Spring RESTful API 개발: 전체 코드 및 개념 총정리 **

오늘은 스프링 부트를 사용하여 **RESTful API**를 구축하는 방법을 학습했다. REST API의 핵심인 **CRUD(Create, Read, Update, Delete)** 연산을 HTTP 메서드에 매핑하고, \*\*`@PathVariable`\*\*과 \*\*`@RequestParam`\*\*을 통해 클라이언트의 요청 데이터를 처리하는 방법을 익혔다. 마지막으로, 자바스크립트를 이용해 프론트엔드와 백엔드 API를 연동하는 전체 과정을 실습했다.

-----

## **1. 핵심 어노테이션 정리**

이번 학습에 사용된 스프링의 주요 어노테이션 역할은 다음과 같다.

| 어노테이션 | 역할 및 설명 |
| :--- | :--- |
| **`@Controller`** | **View 반환용 컨트롤러**: 주로 JSP 같은 뷰 템플릿 파일을 찾아 렌더링한 결과를 반환한다. |
| **`@RestController`** | **데이터 반환용 컨트롤러**: `@Controller`와 `@ResponseBody`가 합쳐진 형태로, 메서드의 반환 값을 JSON 같은 데이터 형태로 변환하여 클라이언트에게 직접 응답한다. |
| **`@ResponseBody`** | `@Controller` 클래스의 메서드에 붙여, View를 찾지 않고 반환값을 데이터 자체로 응답하도록 강제한다. |
| **`@RequestMapping`** | URL 요청을 특정 컨트롤러나 메서드에 연결(매핑)한다. 클래스 레벨에 붙여 공통 URL을 처리할 수 있다. |
| **`@GetMapping`** | `GET` 방식의 HTTP 요청을 처리하는 메서드에 사용되는 축약형 매핑 어노테이션. |
| **`@PostMapping`** | `POST` 방식의 HTTP 요청을 처리하는 메서드에 사용되는 축약형 매핑 어노테이션. |
| **`@PutMapping`** | `PUT` 방식의 HTTP 요청을 처리하는 메서드에 사용되는 축약형 매핑 어노테이션. |
| **`@DeleteMapping`** | `DELETE` 방식의 HTTP 요청을 처리하는 메서드에 사용되는 축약형 매핑 어노테이션. |
| **`@RequestParam`** | URL의 쿼리 파라미터(`?key=value`) 값을 메서드의 파라미터 변수에 바인딩한다. |
| **`@PathVariable`** | URL 경로의 일부(`/books/{id}`)를 파라미터 변수에 바인딩한다. |

-----

## **2. 도서 관리 API 구현 및 프론트엔드 연동**

'도서' 자원에 대한 CRUD API를 구현하고, 자바스크립트 클라이언트와 연동하는 과정이다.

### **2-1. Controller: `BookController2_4.java`**

* **개념 분석**: 이 컨트롤러는 '도서(Book)'라는 자원에 대해 CRUD 연산을 처리한다. 각 HTTP 메서드(`GET`, `POST`, `PUT`, `DELETE`)가 명확한 역할(조회, 생성, 수정, 삭제)을 수행하며, 이는 RESTful API 설계의 기본 원칙을 따른다.
* **주요 어노테이션**:
    * `@RestController`: 이 클래스의 모든 메서드가 기본적으로 JSON 데이터를 반환하도록 설정한다.
    * `@RequestMapping("/api/v2-4/books")`: 이 컨트롤러의 모든 요청 URL은 `/api/v2-4/books`로 시작하도록 공통 경로를 지정한다.

<!-- end list -->

```java
package com.spring.basic.chap2_4.controller;

import com.spring.basic.chap2_4.entity.Book;
import org.springframework.web.bind.annotation.*;

import java.util.*;
import java.util.stream.Collectors;

// 이 클래스가 REST API를 처리하는 컨트롤러임을 명시
// 각 메서드에 @ResponseBody를 붙인 것과 동일하게 동작
@RestController
// 이 컨트롤러의 모든 요청 URL은 "/api/v2-4/books"로 시작함
@RequestMapping("/api/v2-4/books")
public class BookController2_4 {

    // 데이터베이스 대용으로 책들을 모아서 관리
    Map<Long, Book> bookStore = new HashMap<>();

    // 책 ID를 순차적으로 생성하기 위한 변수
    private Long nextId = 1L;

    // 컨트롤러 객체 생성 시 초기 데이터 3개를 저장
    public BookController2_4() {
        bookStore.put(nextId, new Book(nextId, "클린코드", "로버트 마틴", 20000));
        nextId++;
        bookStore.put(nextId, new Book(nextId, "해리포터", "조앤 롤링", 10000));
        nextId++;
        bookStore.put(nextId, new Book(nextId, "삼국지", "나관중", 15000));
        nextId++;
    }

    // 1. 목록 조회 (GET /api/v2-4/books)
    /*
        ?sort=[ id | title | price ]
        - 기본값은 id 오름차, title은 가나다순, price는 내림차
     */
    @GetMapping
    public Map<String, Object> list(@RequestParam(defaultValue = "id") String sort) {
        // Map의 value들(Book 객체들)을 리스트로 변환
        List<Book> bookList = new ArrayList<>(bookStore.values())
                .stream()
                .sorted(getComparator(sort)) // 정렬 기준에 따라 정렬
                .collect(Collectors.toList());

        // 총 권수와 책 목록을 Map에 담아 JSON으로 반환
        return Map.of(
                "count", bookStore.size(),
                "bookList", bookList
        );
    }

    // 정렬 기준을 동적으로 생성하는 도우미 메서드
    private Comparator<Book> getComparator(String sort) {
        Comparator<Book> comparing = null;
        switch (sort) {
            case "title": // 제목 기준 오름차순
                comparing = Comparator.comparing(Book::getTitle);
                break;
            case "price": // 가격 기준 내림차순
                comparing = Comparator.comparing(Book::getPrice).reversed();
                break;
            default: // id 기준 오름차순 (기본값)
                comparing = Comparator.comparing(Book::getId);
        }
        return comparing;
    }


    // 2. 개별 조회 (GET /api/v2-4/books/{id})
    @GetMapping("/{id}")
    public Book getBook(@PathVariable Long id) { // URL 경로의 {id} 값을 읽음
        Book foundBook = bookStore.get(id);
        return foundBook;
    }

    // 3. 도서 생성 (POST /api/v2-4/books)
    @PostMapping
    public String createBook(String title, String author, int price) { // 쿼리 파라미터를 자동으로 매핑
        // 새 도서 객체 생성
        Book book = new Book(nextId++, title, author, price);

        // 맵에 저장
        bookStore.put(book.getId(), book);

        return "도서 추가 완료: " + book.getId();
    }

    // 4. 도서 삭제 (DELETE /api/v2-4/books/{id})
    // 삭제 요청   /api/v2-4/books/99  -> 삭제실패 메시지 응답
    @DeleteMapping("/{id}")
    public String deleteBook(@PathVariable Long id) {
        Book removed = bookStore.remove(id);
        if (removed == null) {
            return id + "번 도서는 존재하지 않습니다. 삭제 실패!";
        }
        return "도서 삭제 완료! - " + id;
    }

    // 5. 도서 수정 (PUT /api/v2-4/books/{id})
    @PutMapping("/{id}")
    public String updateBook(
            String title,
            String author,
            int price,
            @PathVariable Long id
    ) {
        Book foundBook = bookStore.get(id);
        if (foundBook == null) {
            return id + "번 도서는 존재하지 않습니다.";
        }
        foundBook.updateBookInfo(title, author, price);
        return "도서 수정 완료: id - " + id;
    }

    // 책이 몇권 저장됐는지 알려주기
    @GetMapping("/count")
    public String count() {
        return "현재 저장된 도서의 개수: " + bookStore.size() + "권";
    }
}
```

### **2-2. Client: `book.js`**

* **개념 분석**: 클라이언트 측의 자바스크립트는 `fetch` API를 사용하여 백엔드 컨트롤러와 통신하고, 그 결과를 받아 화면을 동적으로 렌더링한다. `async/await`을 통해 비동기 통신을 동기적인 코드처럼 쉽게 작성할 수 있다.

<!-- end list -->

```javascript
// 백엔드 API 서버의 기본 URL
const URL = '/api/v2-4/books';

//=========== 렌더링 관련 함수 ============//

// 책 목록을 화면에 렌더링하는 함수
const renderBooks = ({bookList, count}) => {
    // 1. 책 목록을 담을 ul 태그와 총 권수를 표시할 태그를 가져옵니다.
    const $bookList = document.querySelector('.book-list');
    const $bookCount = document.querySelector('.book-count');

    // 2. 렌더링을 시작하기 전에 기존 목록을 전부 비웁니다.
    //    이 과정이 없으면 목록을 새로고침할 때마다 책들이 중복되어 나타납니다.
    $bookList.innerHTML = '';

    // 3. 총 권수를 업데이트합니다.
    $bookCount.textContent = count;

    // 4. 받아온 책 목록(bookList)을 순회하면서 li 태그를 생성합니다.
    bookList.forEach(book => {
        const $newLi = document.createElement('li');
        // 생성된 li 태그에 data-id 속성으로 책의 고유 ID를 부여합니다. (삭제 시 사용)
        $newLi.dataset.id = book.id;

        $newLi.innerHTML = `
      <div class="info">
          <h3>${book.title}</h3>
          <p>${book.author}</p>
      </div>
      <div class="price">
          <span>${book.price}원</span>
          <button class="del-btn">삭제</button>
      </div>
    `;
        $bookList.append($newLi);
    });
};


//=========== 서버 데이터 요청/응답 관련 함수 ============//

// 서버에서 책 목록을 가져오는 비동기 함수
const fetchGetBooks = async (sort = 'id') => {
    // 정렬 파라미터를 URL에 추가합니다. 기본값은 'id'입니다.
    const res = await fetch(`${URL}?sort=${sort}`);
    const result = await res.json();
    // 받아온 데이터로 렌더링 함수를 호출합니다.
    renderBooks(result);
};

// 서버에 책을 등록하는 비동기 함수
const fetchPostBook = async () => {
    // 1. 폼에서 사용자가 입력한 제목, 저자, 가격 정보를 가져옵니다.
    const $titleInput = document.getElementById('title');
    const $authorInput = document.getElementById('author');
    const $priceInput = document.getElementById('price');

    // 2. 입력값이 모두 채워졌는지 간단히 확인합니다.
    if (!$titleInput.value.trim() || !$authorInput.value.trim() || !$priceInput.value) {
        alert('모든 필드를 채워주세요!');
        return;
    }

    // 3. 서버에 보낼 데이터를 객체로 만듭니다.
    const payload = {
        title: $titleInput.value,
        author: $authorInput.value,
        price: +$priceInput.value, // '+'를 붙여 숫자 타입으로 변환
    };

    // 4. fetch를 사용하여 POST 요청을 보냅니다.
    await fetch(`${URL}?title=${payload.title}&author=${payload.author}&price=${payload.price}`, {
        method: 'POST'
    });

    // 5. 등록이 완료된 후, 입력창을 비우고 목록을 새로고침합니다.
    $titleInput.value = '';
    $authorInput.value = '';
    $priceInput.value = '';

    await fetchGetBooks(); // 목록 새로고침
};


// 서버에 책 삭제를 요청하는 비동기 함수
const fetchDeleteBook = async (id) => {
    // 정말 삭제할 것인지 사용자에게 한 번 더 확인합니다.
    if (!confirm(`${id}번 도서를 정말로 삭제하시겠습니까?`)) {
        return;
    }

    // fetch를 사용하여 DELETE 요청을 보냅니다.
    const res = await fetch(`${URL}/${id}`, {
        method: 'DELETE',
    });

    await fetchGetBooks(); // 목록 새로고침
};


//=========== 이벤트 핸들러 설정 ============//

// 각종 이벤트(클릭, 제출 등)를 감지하고 그에 맞는 함수를 호출하는 핸들러
const addEventListeners = () => {
    const $bookForm = document.getElementById('book-form');
    const $sortButtons = document.querySelector('.sort-buttons');
    const $bookList = document.querySelector('.book-list');

    // 1. 도서 등록 폼 제출(submit) 이벤트
    $bookForm.addEventListener('submit', (e) => {
        // 폼 제출 시 브라우저가 새로고침되는 기본 동작을 막습니다.
        e.preventDefault();
        fetchPostBook();
    });

    // 2. 정렬 버튼 클릭 이벤트 (이벤트 위임 사용)
    $sortButtons.addEventListener('click', (e) => {
        // 클릭된 요소가 .sort-btn 클래스를 가지고 있는지 확인
        if (!e.target.matches('.sort-btn')) {
            return;
        }
        // 클릭된 버튼의 data-sort 속성 값을 가져와 fetch 함수에 전달
        fetchGetBooks(e.target.dataset.sort);
    });

    // 3. 삭제 버튼 클릭 이벤트 (이벤트 위임 사용)
    $bookList.addEventListener('click', (e) => {
        if (!e.target.matches('.del-btn')) {
            return;
        }
        // 클릭된 삭제 버튼에서 가장 가까운 li 태그를 찾아 data-id 값을 가져옵니다.
        const bookId = e.target.closest('li').dataset.id;
        fetchDeleteBook(bookId);
    });
};


//=========== 메인 코드 실행 ============//
(function () {
    // 페이지가 처음 열렸을 때 기본 목록(id순)을 불러옵니다.
    fetchGetBooks();

    // 이벤트 핸들러들을 등록합니다.
    addEventListeners();
})();
```

-----

## **3. 추가 학습: 다양한 요청과 응답 처리**

### **3-1. `@PathVariable` vs. `@RequestParam` 심화 (`PracticeController2_3.java`)**

* **개념**: 두 어노테이션은 클라이언트의 데이터를 받는다는 점은 같지만, 데이터의 위치가 다르다. `@PathVariable`은 URL 경로의 일부를, `@RequestParam`은 `?` 뒤에 오는 쿼리 스트링의 값을 읽는다.

<!-- end list -->

```java
package com.spring.basic.chap2_3.Practice;

import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/practice/api/v1")
public class PracticeController2_3 {

    @GetMapping("/welcome")
    public String welcome() {
        return "welcome to spring MVC!";
    }

    // @PathVariable 사용 예시: /practice/api/v1/product/123
    // URL 경로 자체에 포함된 값을 읽는다.
    @GetMapping("/product/{id}")
    public String getProduct(@PathVariable("id") String ProductId) {
        return "product Id = " + ProductId;
    }

    // @RequestParam 사용 예시: /practice/api/v1/books?author=kim&genre=comic
    // '?' 뒤에 오는 쿼리 스트링의 값을 읽는다.
    @GetMapping("/books")
    public String getBook(
            @RequestParam("author") String author,
            @RequestParam("genre") String genre
    ){
        return "author = " + author + ", genre = " + genre;
    }

    @GetMapping("/search")
    public String search(
            @RequestParam("query") String query,
            @RequestParam(value = "value", defaultValue = "1") int page)
    { return "Query: " + query + "page =" + page;
    }


    @GetMapping("/info/{userId}")
    public String getInfo(@PathVariable("userId") String userId) {
        return "User Info: " + userId;
    }
}
```

### **3-2. 다양한 데이터 형식 응답 (`ResponseController2_5.java`)**

* **개념**: `@RestController` 또는 `@ResponseBody`가 붙은 메서드는 Spring 내부의 `HttpMessageConverter`가 반환되는 자바 객체(List, Map, 사용자 정의 객체 등)를 자동으로 JSON 문자열로 변환하여 응답 본문에 담아준다.

<!-- end list -->

```java
package com.spring.basic.chap2_5.controller;

import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.ToString;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Set;

@Controller
@RequestMapping("/api/v2-5")
public class ResponseController2_5 {

    // 페이지 라우팅 - 특정 뷰 (JSP, thymeleaf)를 포워딩 해주는 것
    @GetMapping("/book-page")
    public String bookPage() {
        return "book-page";
    }

    // 클라이언트에게 데이터 응답
    // html 응답
    @GetMapping("/show/html")
    @ResponseBody // 순수 데이터를 클라이언트에게 전달
    public String html() {
        return """
                <html>
                <body>
                    <h1>HTML 응답하기</h1>
                </body>
                </html>
                """;
    }

    // 순수 텍스트 응답하기
    @GetMapping(value="/show/text", produces = "text/plain")
    @ResponseBody // 2가지 : JSON이거나 HTML
    public String text() {
        return "하이 난 문자야~~";
    }

    // 순수 텍스트 응답하기2
    @GetMapping(value="/show/text2", produces = "application/json")
    @ResponseBody
    public Map<String, Object> text2() {
        return Map.of(
                "message", "하이 난 문자야2~~2"
        );
    }

    // JSON배열 응답 - 자바 배열이나 리스트, 셋
    @GetMapping("/json/array")
    @ResponseBody
    public List<String> hobbies() {
        return List.of("테니스", "수학문제풀기", "탁구");
    }

    // JSON 객체 응답 - 자바의 클래스의 인스턴스(객체) or Map
    @GetMapping("/json/object")
    @ResponseBody
    public Map<String, Object> myPet() {
        return Map.of(
                "name", "야옹이",
                "age", 3,
                "kind", "코리안숏헤어"
        );
    }

    @GetMapping("/json/object2")
    @ResponseBody
    public Pet myPet2() {
        return new Pet("냥냥이", 5, "페르시안", true);
    }

    @GetMapping("/json/object3")
    @ResponseBody
    public List<Pet> myPet3() {
        return List.of(
                new Pet("냥냥이", 5, "페르시안", true),
                new Pet("냥냥이2", 6, "페르시안2", false),
                new Pet("냥냥이3", 7, "페르시안3", true),
                new Pet("냥냥이4", 8, "페르시안4", false)
        );
    }

    @ToString @Getter
    @AllArgsConstructor
    public static class Pet {
        private String name;
        private int age;
        private String kind;
        private boolean injection;
    }
}
```

-----

## **4. 데이터 모델**

### **`Book.java`**

* **개념**: `Lombok` 라이브러리를 사용하면 `@Getter`, `@Setter` 등의 어노테이션만으로 반복적인 상용구 코드를 자동으로 생성할 수 있어 코드가 매우 간결해진다.

<!-- end list -->

```java
package com.spring.basic.chap2_4.entity;

import lombok.*;

// Lombok 어노테이션: 컴파일 시 자동으로 getter, setter 등을 생성
@Getter
@Setter
@EqualsAndHashCode
@NoArgsConstructor // 기본 생성자
@AllArgsConstructor // 전체 필드 초기화 생성자
@ToString
public class Book {

    // long타입이 아닌 Long을 쓰는 이유 : null 과 0의 차이 null은 가격자체가 설정 안된 것 , 0은 그냥 0원임
    private Long id; // 책을 유일하게 구분할 수 있는 식별자
    // @Setter 타이틀만 setter 쓸 수 있음
    private String title;
    private String author;
    private int Price;


    // 수정 편의 메서드
    public void updateBookInfo(String title, String author, int price) {
        this.title = title;
        this.author = author;
        this.Price = price;
    }
}
```