
# **Spring JdbcTemplate: JDBC를 더 편리하게**

오늘은 지난 시간에 학습한 순수 JDBC의 상용구 코드(boilerplate code)를 스프링의 \*\*`JdbcTemplate`\*\*을 사용하여 대폭 개선하는 방법을 학습했다. `JdbcTemplate`은 커넥션 관리, 예외 처리, 리소스 반납 등 반복적인 작업을 대신 처리해주어, 개발자가 오직 **SQL 작성과 결과 처리**에만 집중할 수 있도록 돕는다.

-----

## **1. `JdbcTemplate`의 역할과 장점**

* **템플릿/콜백 패턴**: `JdbcTemplate`은 템플릿/콜백 패턴을 사용하여, 반복되는 JDBC 작업(템플릿)은 자신이 알아서 처리하고, 개발자가 정의한 작업(콜백)만 수행하도록 구조화되어 있다.
* **주요 장점**:
    1.  **리소스 관리 자동화**: `Connection`, `PreparedStatement`, `ResultSet` 등의 객체를 자동으로 생성하고 `close()` 해준다. (`try-with-resources` 불필요)
    2.  **예외 처리**: JDBC의 `SQLException`(Checked Exception)을 스프링의 `DataAccessException`(Unchecked Exception)으로 변환하여, 번거로운 `try-catch` 작성을 줄여준다.
    3.  **간결한 코드**: `update()`, `query()`, `queryForObject()` 등 직관적인 메서드를 제공하여 CRUD 작업을 매우 간결하게 작성할 수 있다.

-----

## **2. `JdbcTemplate` 핵심 메서드 정리**

| 메서드 | 설명 | 반환 타입 | 사용처 |
| :--- | :--- | :--- | :--- |
| **`update()`** | `INSERT`, `UPDATE`, `DELETE` 등 데이터 변경 SQL을 실행한다. | `int` (적용된 행의 수) | `save()`, `update()`, `delete()` |
| **`query()`** | `SELECT` 결과를 `List` 형태로 반환한다. (결과가 0개 이상일 때) | `List<T>` | `findAll()` |
| **`queryForObject()`**| `SELECT` 결과를 단일 객체로 반환한다. (결과가 반드시 1개일 때) | `T` (지정된 객체 타입) | `findById()`, 집계 함수 조회 |

-----

## **3. `JdbcTemplate`을 이용한 Repository 구현 (`BookSpringRepository`)**

순수 JDBC로 구현했던 `BookRepository`를 `JdbcTemplate`으로 리팩토링한 예시다.

### **3-1. `BookSpringRepository.java` 전체 코드 및 분석**

```java
package com.spring.database.chap01.repository;

import com.spring.database.chap01.entity.Book;
import lombok.RequiredArgsConstructor;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Repository;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.List;

// Spring JDBC로 도서 CRUD를 관리
@Repository("bsr") // "bsr"이라는 이름의 빈으로 등록
@RequiredArgsConstructor // final 필드 생성자 주입
public class BookSpringRepository implements BookRepository{

    // JdbcTemplate 객체는 스프링이 DB 설정(yml)을 읽어 자동으로 생성 및 주입해준다.
    private final JdbcTemplate template;

    @Override
    public boolean save(Book book) {
        String sql = """
                    INSERT INTO BOOKS
                     (title, author, isbn)
                      VALUES
                       (?,?,?)
                    """;
        // update() 메서드는 INSERT, UPDATE, DELETE 구문에 사용
        // 파라미터로 (SQL, ?에 순서대로 들어갈 값...)을 전달
        return template.update(
                sql,
                book.getTitle(),
                book.getAuthor(),
                book.getIsbn()
        ) == 1; // 성공한 행의 수가 1이면 true 반환
    }

    @Override
    public boolean updateTitleAndAuthor(Book book) {
        String sql = """
                    UPDATE BOOKS
                    SET author = ?, title = ?
                    WHERE id = ?
                    """;
        return template.update(
                sql,
                book.getAuthor(),
                book.getTitle(),
                book.getId()
        ) == 1;
    }

    @Override
    public boolean deleteById(Long id) {
        String sql = """
                    DELETE FROM BOOKS
                    WHERE id = ?
                    """;

        return template.update(sql,id) == 1;
    }

    @Override
    public List<Book> findAll() {
        String sql = """
                    SELECT * FROM books
                    """;
        // query() 메서드는 SELECT 결과를 List 형태로 반환
        // 두 번째 파라미터는 RowMapper 객체로,
        // 조회된 각 행(row)을 어떤 객체(Book)에 매핑할지 방법을 정의
        return template.query(sql, (ResultSet rs, int rowNum) -> new Book(rs));
    }

    @Override
    public Book findById(Long id) {
        String sql = """
                    SELECT * FROM books
                    WHERE id = ?
                    """;
        // queryForObject()는 단 하나의 결과만 반환할 때 사용
        // 결과가 없거나 2개 이상이면 예외 발생
        return template.queryForObject(sql,(ResultSet rs, int n) -> new Book(rs),id);
    }
}
```

* **분석**: 모든 메서드가 `try-catch`나 `Connection` 관리 코드 없이, `template`의 메서드를 호출하는 단 한 줄의 코드로 간결하게 바뀌었다.

-----

## **4. `RowMapper` 심화 및 DTO 매핑 (`ProductRepository`)**

`RowMapper`는 데이터베이스 조회 결과(`ResultSet`)의 각 행을 어떻게 자바 객체로 변환할지 알려주는 명세서다.

### **4-1. `ProductRepository.java` 전체 코드 및 분석**

```java
package com.spring.database.chap02.repository;

import com.spring.database.chap02.dto.PriceInfo;
import com.spring.database.chap02.entity.Product;
import lombok.RequiredArgsConstructor;
import org.springframework.jdbc.core.BeanPropertyRowMapper;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Repository;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.List;
import java.util.Map;

@Repository
@RequiredArgsConstructor
public class ProductRepository {

    private final JdbcTemplate template;

    // 상품의 기본적인 CRUD를 5개
    // 생성, 수정, 삭제(논리삭제), 전체조회, 단일조회

    public void save(Product product) {
        String sql = """
                INSERT INTO PRODUCTS
                    (name, price, stock_quantity, description, seller)
                VALUES
                    (?, ?, ?, ?, ?)
                """;
        template.update(sql,
                product.getName(), product.getPrice(),
                product.getStockQuantity(), product.getDescription(),
                product.getSeller()
        );
    }

    public void update(Product product) {
        String sql = """
                UPDATE PRODUCTS
                SET name = ?, price = ?, stock_quantity =?,
                      description = ?, seller = ?
                WHERE id = ?
                """;
        template.update(
                sql, product.getName(), product.getPrice()
                , product.getStockQuantity(), product.getDescription()
                , product.getSeller(), product.getId()
        );
    }

    // 논리적 삭제: 실제 데이터를 삭제하는 대신, 상태(status)를 변경하여 삭제된 것처럼 처리
    public void deleteById(Long id) {
        String sql = """
                UPDATE PRODUCTS
                SET status = 'DELETED'
                WHERE id = ?
                """;
        template.update(sql, id);
    }

    // 전체조회 - 논리삭제된 행을 제외해야 함
    public List<Product> findAll() {
        String sql = """
                SELECT * FROM PRODUCTS
                WHERE status <> 'DELETED'
                """;

        // BeanPropertyRowMapper: 테이블의 컬럼명과 엔터티클래스의 필드명이
        // 똑같을 경우 (snake_case ↔ camelCase 자동 변환 포함) 자동 매핑해줌
        return template.query(sql, new BeanPropertyRowMapper<>(Product.class));
    }

    // 전체 상품의 총액과 평균가격을 가져오는 기능
    public PriceInfo getPriceInfo() {
        String sql = """
                SELECT
                   SUM(price) AS total_price
                    , AVG(price) AS average_price
               FROM PRODUCTS
               WHERE status = 'ACTIVE'
               """;
        // 집계 함수 결과는 일반 Entity에 매핑하기 어려우므로,
        // DTO(PriceInfo)에 매핑하기 위한 RowMapper를 람다식으로 직접 구현
        return template.queryForObject(sql,
                (rs,  rowNum) -> {
                    int totalPrice = rs.getInt("total_price");
                    double averagePrice = rs.getDouble("average_price");

                    return new PriceInfo(totalPrice, averagePrice);
                });
    }

}
```

### **4-2. `RowMapper`의 종류 및 활용**

* **람다식을 이용한 직접 구현**: `(rs, rowNum) -> new Book(rs)` 처럼, `RowMapper` 인터페이스를 람다식으로 직접 구현할 수 있다. `SUM`, `AVG` 같은 집계 함수 결과를 DTO에 담을 때 유용하다.
* **`BeanPropertyRowMapper`**: DB 컬럼명(`stock_quantity`)과 자바 필드명(`stockQuantity`)이 규칙에 맞게 일치할 경우, **자동으로** `ResultSet`을 해당 자바 빈(Bean) 객체에 매핑해주는 매우 편리한 `RowMapper` 구현체다.

-----

## **5. 관련 DTO 및 Entity 코드**

### **`Product.java`**

```java
package com.spring.database.chap02.entity;

/*
CREATE TABLE products (
        id BIGINT AUTO_INCREMENT,
        name VARCHAR(100) NOT NULL COMMENT '상품명',
        price INT NOT NULL COMMENT '가격',
        stock_quantity INT NOT NULL COMMENT '재고수량',
        description TEXT COMMENT '상품설명',
        seller VARCHAR(50) NOT NULL COMMENT '판매자',
        status VARCHAR(20) DEFAULT 'ACTIVE' COMMENT '상품상태',
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='상품';
*/

import lombok.*;
import java.time.LocalDateTime;

@EqualsAndHashCode
@Getter @Setter @ToString
@AllArgsConstructor @NoArgsConstructor
@Builder
public class Product {

    private Long id;
    private String name;
    private int price;
    private int stockQuantity;
    private String description;
    private String seller;

    // ACTIVE : 삭제되지 않은 것 , DELETED: 제거된 상품 (논리적 삭제)
    private String status;
    private LocalDateTime createdAt;
}
```

### **`PriceInfo.java` (DTO)**

```java
package com.spring.database.chap02.dto;

import lombok.*;

@Getter @Setter @ToString
@EqualsAndHashCode
@NoArgsConstructor @AllArgsConstructor
@Builder
public class PriceInfo {

    private int totalPrice;
    private double averagePrice;
}
```