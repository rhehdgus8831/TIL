# **Spring과 순수 JDBC 연동**

오늘은 스프링 부트 애플리케이션에서 데이터베이스에 연결하고, Java의 표준 DB 연동 기술인 **JDBC**를 사용하여 직접 SQL을 실행하는 방법을 학습했다. 이를 통해 `DataSource`, `Connection`, `PreparedStatement`, `ResultSet` 등 JDBC의 핵심 객체들의 역할과 전체적인 데이터 처리 흐름을 이해한다.

-----

## **1. 데이터베이스 연동 설정 (`application.yml`)**

스프링 부트는 `application.yml` 파일에 데이터베이스 연결 정보를 명시하면, 자동으로 **`DataSource`** 객체를 생성하여 스프링 컨테이너에 빈으로 등록해준다.

* **`DataSource`**: 데이터베이스 커넥션 풀을 관리하는 객체. 필요할 때마다 커넥션 객체를 제공하고 반납받아, 효율적인 DB 연결을 가능하게 한다.

<!-- end list -->

```yaml
server:
  port: 9001

spring:
  application:
    name: spring-db202507

  # Spring DB 연동 !필수
  # 이 정보를 바탕으로 스프링 부트가 DataSource 객체를 자동 생성 및 설정
  datasource:
    # DB 서버 접속 URL (드라이버, 주소, 포트, DB명 포함)
    url: jdbc:mariadb://localhost:3306/spring_study
    # DB 접속 계정
    username: root
    # DB 접속 비밀번호
    password: mariadb

logging:
  level:
    root: INFO
    # com.spring.database 패키지 하위의 로그는 DEBUG 레벨까지 모두 출력
    com.spring.database: DEBUG
```

-----

## **2. JDBC 핵심 객체와 CRUD 동작 흐름**

순수 JDBC를 사용한 데이터베이스 작업은 보통 다음과 같은 흐름으로 진행된다.

1.  **`DataSource`에서 `Connection` 얻기**: `dataSource.getConnection()`을 호출하여 DB 커넥션을 획득한다.
2.  **SQL 문 준비**: 실행할 SQL 쿼리를 문자열로 작성한다.
3.  **`PreparedStatement` 생성**: `conn.prepareStatement(sql)`를 통해 SQL을 실행할 객체를 생성한다. `PreparedStatement`는 SQL 인젝션 공격을 방지하고, 반복 실행 시 성능상 이점이 있다.
4.  **`?` 파라미터 채우기**: `pstmt.set타입(인덱스, 값)` 메서드로 SQL의 `?` 부분을 실제 값으로 채운다. **인덱스는 1부터 시작한다.**
5.  **SQL 실행**:
    * `INSERT`, `UPDATE`, `DELETE`: `pstmt.executeUpdate()` 호출. (결과: 적용된 행의 수)
    * `SELECT`: `pstmt.executeQuery()` 호출. (결과: `ResultSet` 객체)
6.  **결과 처리 (SELECT의 경우)**: `ResultSet` 객체의 `rs.next()`를 `while`문으로 반복하여 조회된 데이터를 한 행씩 처리한다.
7.  **리소스 반납**: `Connection`, `PreparedStatement`, `ResultSet` 등 사용한 객체는 반드시 `close()` 해야 한다. **`try-with-resources`** 구문을 사용하면 자동으로 처리되어 편리하다.

-----

## **3. `BookTest.java` 전체 코드 및 분석**

`@SpringBootTest`를 사용하면 테스트 환경에서도 스프링 컨테이너가 동작하여, `@Autowired`로 `DataSource` 같은 빈을 주입받아 테스트할 수 있다.

```java
package com.spring.database.chap01.entity;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;

@SpringBootTest // 의존 객체를 테스트시에 주입받기 위한 설정
class BookTest {

    @Autowired // 필드 주입: 스프링 컨테이너에 등록된 DataSource 빈을 자동으로 주입받음
    private DataSource dataSource;

    @Test
    void insertTest() {
        // 책 한권의 데이터를 DB에 실제로 저장
        // DB를 연결
        
        // try-with-resources: 이 블록이 끝나면 conn 객체가 자동으로 close() 됨
        try (Connection conn = dataSource.getConnection()){

            // 1. 데이터베이스 소켓 연결 - DB에 인증정보를 주고 확인받는 작업
            System.out.println("conn = " + conn);

            // 2. SQL 작성
            String sql = """
                    INSERT INTO BOOKS
                     (title, author, isbn)
                      VALUES
                       (?,?,?)
                    """;
            // 3. SQL을 실행하는 객체를 가져와야 함.
            PreparedStatement pstmt = conn.prepareStatement(sql);

            // 4. 실행하기전에 ? 값들을 채우는 작업
            pstmt.setString(1,"사랑의 하츄핑"); // SQL은 인덱스가 1번부터 시작함
            pstmt.setString(2,"캐치캐치 티니핑");
            pstmt.setString(3,"B001");

            // 5. SQL 실행 (INSERT, UPDATE, DELETE는 executeUpdate 사용)
            pstmt.executeUpdate();

        } catch (Exception e) {
            e.printStackTrace();
        }
        /*finally {
            // 데이터베이스 연결 해제 (try-with-resources 구문이 이 코드를 대체함)
            try {
                if (conn != null) conn.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }*/
    }

    @Test
    void updateTest() {
        try(Connection conn = dataSource.getConnection()){

            // 2번 책의 저자와 ISBN을 수정하는 SQL
            String sql = """
                    UPDATE BOOKS
                    SET author = ?, isbn = ?
                    WHERE id = ?
                    """;

            PreparedStatement pstmt = conn.prepareStatement(sql);

            // ? 파라미터 채우기
            pstmt.setString(1,"오로라핑");
            pstmt.setString(2,"Z999");
            pstmt.setLong(3,2L);

            pstmt.executeUpdate();

        }catch (Exception e){
            e.printStackTrace();
        }
    }

    @Test
    void deleteTest(){
        try(Connection conn = dataSource.getConnection()){
            // 2번 책을 삭제하는 SQL
            String sql = """
                    DELETE FROM BOOKS
                    WHERE id = ?
                    """;

            PreparedStatement pstmt = conn.prepareStatement(sql);
            pstmt.setLong(1,2L);
            pstmt.executeUpdate();

        }catch (Exception e){
            e.printStackTrace();
        }
    }

    // 전체조회
    @Test
    void findAllTest(){
        try(Connection conn = dataSource.getConnection()) {

            String sql = """
                    SELECT * FROM BOOKS
                    WHERE title LIKE ?
                    """;
            PreparedStatement pstmt = conn.prepareStatement(sql);
            pstmt.setString(1,"%잼%");

            // SQL 실행 (SELECT는 executeQuery 사용)
            // 조회 결과를 ResultSet 객체로 받음
            ResultSet rs = pstmt.executeQuery();

            // rs.next()는 다음 행에 데이터가 있으면 true, 없으면 false
            // 반복문을 통해 조회된 모든 행을 순회
            while(rs.next()){
                // 현재 행의 컬럼 데이터를 타입에 맞게 읽어옴
                String title = rs.getString("title");
                String author = rs.getString("author");
                String isbn = rs.getString("isbn");

                System.out.println("title = " + title);
                System.out.println("author = " + author);
                System.out.println("isbn = " + isbn);
                System.out.println("================================");
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

-----

## **4. 데이터베이스 테이블 및 모델**

### **`Book.java` (Entity)**

* **분석**: 데이터베이스의 `books` 테이블과 1:1로 매핑되는 자바 객체.

<!-- end list -->

```java
package com.spring.database.chap01.entity;

import lombok.*;
import java.time.LocalDateTime;

@EqualsAndHashCode
@Getter @Setter @ToString
@AllArgsConstructor @NoArgsConstructor
@Builder
public class Book {
    
    private Long id;
    private String title;
    private String author;
    private String isbn;
    private boolean available;
    private LocalDateTime createdAt;
}
```

### **SQL DDL (테이블 생성문)**

```sql
-- books 테이블 생성
CREATE TABLE IF NOT EXISTS books (
                                     id BIGINT AUTO_INCREMENT,
                                     title VARCHAR(200) NOT NULL COMMENT '도서 제목',
                                     author VARCHAR(100) NOT NULL COMMENT '저자',
                                     isbn VARCHAR(13) NOT NULL COMMENT 'ISBN',
                                     available BOOLEAN DEFAULT true COMMENT '대출 가능 여부',
                                     created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '등록일',
                                     PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='도서 정보';
```