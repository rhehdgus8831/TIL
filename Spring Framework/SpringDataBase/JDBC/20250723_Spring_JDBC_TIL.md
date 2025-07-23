# **JDBC 리팩토링: Repository 패턴과 단위 테스트**

오늘은 지난 시간에 학습한 순수 JDBC 코드를, 역할과 책임에 따라 객체지향적으로 분리하는 **리팩토링**을 진행했다. 데이터베이스와의 통신 로직을 **Repository 계층**으로 분리하고, 이 계층을 **Controller**가 사용하여 **REST API**를 제공했다. 마지막으로, 구현된 Repository의 기능이 올바르게 동작하는지 **JUnit5를 이용한 단위 테스트**로 검증하는 방법을 학습했다.

-----

## **1. Repository 패턴: 데이터 접근 로직의 분리**

### **1-1. 개념 설명**

**리포지토리 패턴(Repository Pattern)** 은 데이터베이스에 접근하여 CRUD를 수행하는 로직을 별도의 클래스로 분리하는 디자인 패턴이다.

* **장점**:
    * **관심사 분리**: 데이터베이스 관련 코드가 한 곳에 모여 있어, 비즈니스 로직이나 화면 로직과 섞이지 않는다.
    * **재사용성**: 컨트롤러, 서비스 등 여러 곳에서 데이터베이스 기능이 필요할 때 리포지토리 객체를 주입받아 재사용할 수 있다.
    * **유지보수성**: 데이터베이스 기술이 바뀌거나(e.g., JDBC → JPA) SQL이 변경될 때, 리포지토리 클래스만 수정하면 되므로 유지보수가 용이하다.

### **1-2. `BookRepository.java` 전체 코드 및 분석**

이 클래스는 오직 `books` 테이블에 대한 CRUD 작업에만 집중한다. 이전 `BookTest` 클래스에 흩어져 있던 JDBC 코드를 재사용 가능한 메서드로 캡슐화했다.

```java
package com.spring.database.chap01.repository;

import com.spring.database.chap01.entity.Book;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.util.ArrayList;
import java.util.List;

// @RequiredArgsConstructor: final 필드에 대한 생성자를 자동으로 만들어줌 (생성자 주입)
@RequiredArgsConstructor
// @Repository: 이 클래스가 데이터 접근 계층의 빈(Bean)임을 스프링에게 알림
@Repository
// 역할 : 데이터 베이스에 접근해서 CRUD를 수행하는 객체
public class BookRepository {

    // 스프링으로부터 DataSource 객체를 주입받음 (DB 커넥션 관리)
    private final DataSource dataSource;

    // INSERT 기능 - 도서 생성 기능
    public boolean save(Book book){
        try(Connection conn = dataSource.getConnection()){
            String sql = """
                    INSERT INTO BOOKS (title, author, isbn)
                    VALUES (?,?,?)
                    """;
            PreparedStatement pstmt = conn.prepareStatement(sql);

            pstmt.setString(1,book.getTitle());
            pstmt.setString(2,book.getAuthor());
            pstmt.setString(3,book.getIsbn());

            // executeUpdate()는 INSERT, UPDATE, DELETE에 사용되며,
            // 성공한 행의 수를 반환한다. 1개의 행이 성공적으로 삽입되었으면 1을 반환.
            int result = pstmt.executeUpdate();
            return result == 1;
        }catch (Exception e){
            e.printStackTrace();
            return false;
        }
    }

    // 도서 제목, 저자 수정
    public boolean updateTitleAndAuthor(Book book){
        try(Connection conn = dataSource.getConnection()){
            String sql = """
                    UPDATE BOOKS
                    SET author = ?, title = ?
                    WHERE id = ?
                    """;
            PreparedStatement pstmt = conn.prepareStatement(sql);
            pstmt.setString(1,book.getAuthor());
            pstmt.setString(2,book.getTitle());
            pstmt.setLong(3,book.getId());
            return pstmt.executeUpdate() == 1;
        }catch (Exception e){
            e.printStackTrace();
            return false;
        }
    }

    // 도서 정보 삭제
    public boolean deleteById(Long id){
        try(Connection conn = dataSource.getConnection()){
            String sql = "DELETE FROM BOOKS WHERE id = ?";
            PreparedStatement pstmt = conn.prepareStatement(sql);
            pstmt.setLong(1,id);
            return pstmt.executeUpdate() == 1;
        }catch (Exception e){
            e.printStackTrace();
            return false;
        }
    }

    // 전체 조회 - ORM (Object Relational Mapping)
    public List<Book> findAll() {
        List<Book> bookList = new ArrayList<>();
        try(Connection conn = dataSource.getConnection()){
            String sql = "SELECT * FROM books";
            PreparedStatement pstmt = conn.prepareStatement(sql);
            ResultSet rs = pstmt.executeQuery();
            while (rs.next()){
                // 조회된 각 행(row)의 데이터를 Book 객체로 변환 (수동 ORM)
                bookList.add(new Book(rs));
            }
        }catch (Exception e){
            e.printStackTrace();
        }
        return bookList;
    }

    // ID로 단일 조회 메서드
    public Book findById(Long id){
        try(Connection conn = dataSource.getConnection()){
            String sql = "SELECT * FROM books WHERE id = ?";
            PreparedStatement pstmt = conn.prepareStatement(sql);
            pstmt.setLong(1,id);
            ResultSet rs = pstmt.executeQuery();
            if(rs.next()){
                // 조회된 행이 있다면 Book 객체로 변환하여 반환
                return new Book(rs);
            };
        }catch (Exception e){
            e.printStackTrace();
            return null;
        }
        return null;
    }
}
```

-----

## **2. REST API 계층 구현 (`BookController.java`)**

### **2-1. 개념 설명**

컨트롤러는 HTTP 요청을 받아 해당 요청에 맞는 리포지토리의 메서드를 호출하고, 그 결과를 클라이언트에게 응답하는 역할만 수행한다. 실제 데이터 처리 로직은 모두 리포지토리에 위임한다.

### **2-2. `BookController.java` 전체 코드 및 분석**

```java
package com.spring.database.chap01.api;

import com.spring.database.chap01.entity.Book;
import com.spring.database.chap01.repository.BookRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/books")
@RequiredArgsConstructor
@Slf4j // 로그 출력을 위한 어노테이션
public class BookController {

    // 컨트롤러는 리포지토리에 의존하여 데이터 관련 작업을 처리함
    private final BookRepository bookRepository;

    // 전체 조회 요청: GET /api/v1/books
    @GetMapping
    public ResponseEntity<?> findAll() {
        // 리포지토리에게 전체 조회를 위임하고, 결과를 OK 응답에 담아 반환
        return ResponseEntity.ok(bookRepository.findAll());
    }

    // 생성 요청: POST /api/v1/books
    @PostMapping
    public ResponseEntity<?> create(@RequestBody Book book) {
        // 리포지토리에게 저장을 위임
        bookRepository.save(book);
        return ResponseEntity.ok("도서 등록 성공!");
    }

    // 수정 요청: PUT /api/v1/books/{id}
    @PutMapping("/{id}")
    public ResponseEntity<?> updateBook(@RequestBody Book book) {
        // 리포지토리에게 수정을 위임
        bookRepository.updateTitleAndAuthor(book);
        return ResponseEntity.ok("도서 수정 성공!");
    }

    // 삭제 요청: DELETE /api/v1/books/{id}
    @DeleteMapping("/{id}")
    public ResponseEntity<?> deleteBook(@PathVariable Long id) {
        // 리포지토리에게 삭제를 위임
        bookRepository.deleteById(id);
        return ResponseEntity.ok("도서 삭제 성공!");
    }

    // 개별 조회 요청: GET /api/v1/books/{id}
    @GetMapping("/{id}")
    public ResponseEntity<?> findOne(@PathVariable Long id) {
        // 리포지토리에게 개별 조회를 위임
        return ResponseEntity.ok(bookRepository.findById(id));
    }
}
```

-----

## **3. 단위 테스트(Unit Test)를 통한 기능 검증**

### **3-1. 개념 설명**

**단위 테스트**는 코드의 특정 모듈(단위)이 의도한 대로 정확히 작동하는지 검증하는 절차다. **JUnit**은 자바의 대표적인 테스트 프레임워크다.

* **GWT (Given-When-Then) 패턴**: 테스트 코드를 구조화하는 좋은 방법.
    * **Given**: 테스트에 필요한 데이터와 환경을 설정하는 단계.
    * **When**: 실제로 테스트할 메서드를 실행하는 단계.
    * **Then**: 실행 결과가 예상과 일치하는지 \*\*단언(Assertion)\*\*하는 단계.

### **3-2. `BookRepositoryTest.java` 전체 코드 및 분석**

```java
package com.spring.database.chap01.repository;

import com.spring.database.chap01.entity.Book;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest // 스프링 컨텍스트에서 관리되는 빈을 꺼내올 수 있음
class BookRepositoryTest {

    // 테스트 프레임워크 : JUnit
    // 5 버전부터는 생성자 주입을 막아놈 - 필드 주입 해야함 (테스트에서만)
    @Autowired
    BookRepository bookRepository;

    // 테스트 메서드
    @Test
    // 테스트의 목적을 주석처럼 적는다. 여기서 문장 표현은 단언을 사용한다.
    @DisplayName("도서 정보를 주면 데이터 베이스 book 테이블에 저장된다.")
    void saveTest(){
        // GWT 패턴
        // given - 테스트를 위해 필요한 데이터
        Book givenBook = Book.builder()
                .title("디아블로 4")
                .author("블리자드")
                .isbn("D004")
                .build();

        // when - 실제 테스트가 벌어지는 상황
        boolean flag = bookRepository.save(givenBook);

        // then - 테스트 결과 (단언)
        Assertions.assertTrue(flag); // flag가 true일 것이라고 단언한다.
        System.out.println("flag = " + flag);
    }

    @Test
    @DisplayName("도서의 ID 번호를 주면 도서가 삭제된다")
    void deleteTest() {
        //given
        Long givenId = 6L;

        //when
        boolean flag = bookRepository.deleteById(givenId);

        //then
        assertTrue(flag); // flag가 true일 것이라고 단언한다.
        System.out.println("flag = " + flag);
    }

    @Test
    @DisplayName("전체조회를 하면 도서의 리스트가 반환됨")
    void findAllTest() {
        //given (특별한 입력값 없음)

        //when
        List<Book> bookList = bookRepository.findAll();

        //then
        bookList.forEach(System.out::println);
        
        // bookList의 사이즈가 4일 것이라고 단언한다.
        assertEquals(4,bookList.size());
    }

    @Test
    @DisplayName("적합한 id를 통해 개별조회를 하면 도서 1개의 객체가 반환된다.")
    void findOneTest() {
        //given
        Long givenId = 5L;
        
        //when
        Book foundBook = bookRepository.findById(givenId);

        //then
        System.out.println("foundBook = " + foundBook);
        
        // foundBook이 null이 아닐 것이라고 단언한다.
        assertNotNull(foundBook);
        // foundBook의 제목이 "꿀잼책"일 것이라고 단언한다.
        assertEquals("꿀잼책",foundBook.getTitle());
    }
}
```

-----

## **4. ORM과 보조 코드**

### **4-1. 간단한 ORM 구현 (`Book.java`)**

* **ORM (Object-Relational Mapping)**: 관계형 데이터베이스의 데이터(Row)를 자바 객체(Object)로 자동으로 변환해주는 기술.
* **분석**: `Book` 클래스에 `ResultSet`을 파라미터로 받는 생성자를 추가하여, `findAll()`이나 `findById()`에서 조회된 DB 데이터를 `Book` 객체로 쉽게 변환할 수 있도록 했다. 이는 ORM의 가장 기본적인 형태를 수동으로 구현한 것이다.

<!-- end list -->

```java
package com.spring.database.chap01.entity;

// ... Lombok 어노테이션 생략 ...
import java.sql.ResultSet;
import java.sql.SQLException;
import java.time.LocalDateTime;

public class Book {

    private Long id;
    private String title;
    // ... 필드 생략 ...
    private LocalDateTime createdAt;

    // ResultSet을 받아 Book 객체를 생성하는 생성자
    public Book(ResultSet rs) throws SQLException {
        this.id = rs.getLong("id");
        this.title = rs.getString("title");
        this.author = rs.getString("author");
        this.isbn = rs.getString("isbn");
        this.available = rs.getBoolean("available");
        this.createdAt = rs.getTimestamp("created_at").toLocalDateTime();
    }
}
```

### **4-2. 뷰 라우팅 컨트롤러 (`BookRouteController.java`)**

* **분석**: 이 컨트롤러는 API를 제공하는 `@RestController`와 달리, 순수하게 뷰 페이지(`book-page.html` 등)를 열어주는 역할만 담당하는 `@Controller`다. 이를 통해 API 로직과 페이지 라우팅 로직을 분리할 수 있다.

<!-- end list -->

```java
package com.spring.database.chap01.routes;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

// 타임리프 html 뷰를 포워딩하는 클래스
@Controller
public class BookRouteController {

    @GetMapping("/book-page")
    public String bookPage(){
        return "book-page"; // templates/book-page.html을 열어줌
    }
}
```