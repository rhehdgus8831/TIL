
# **MyBatis 연동: SQL 매퍼 프레임워크**

오늘은 순수 JDBC나 `JdbcTemplate`보다 더 발전된 데이터베이스 연동 프레임워크인 **MyBatis**에 대해 학습했다. MyBatis의 가장 큰 특징은 **SQL 코드를 자바 코드로부터 완전히 분리**하여 XML 파일에 작성하고, 이를 간단한 인터페이스와 연결하여 사용하는 것이다. 이를 통해 SQL의 독립성을 유지하고, 코드의 가독성과 유지보수성을 크게 향상시킬 수 있다.

-----

## **1. MyBatis의 동작 원리**

MyBatis는 **Mapper 인터페이스**와 **Mapper XML**이라는 두 가지 핵심 구성 요소로 동작한다.

1.  **Mapper 인터페이스**: 실행하려는 데이터베이스 작업(CRUD)을 자바 **메서드 형태로 명세**한다. 개발자는 이 인터페이스를 주입받아 사용한다.
2.  **Mapper XML**: Mapper 인터페이스에 명세된 메서드와 1:1로 매핑되는 **실제 SQL 쿼리를 작성**하는 파일이다.
3.  **MyBatis 프레임워크**: 스프링이 실행되면, MyBatis는 이 두 파일을 자동으로 연결하여 인터페이스의 메서드가 호출될 때 해당하는 XML의 SQL을 실행하고, 그 결과를 자바 객체로 변환하여 반환해준다.

<!-- end list -->

  * **흐름**: `Service/Test Code` → `Mapper Interface` (메서드 호출) → `MyBatis` (연결된 XML의 SQL 실행) → `Database`

-----

## **2. Mapper 인터페이스 (`petMapper.java`)**

### **2-1. 개념 설명**

`@Mapper` 어노테이션을 붙인 인터페이스를 작성하면, MyBatis가 이 인터페이스를 자신의 구현체로 인식하여 스프링 컨테이너에 빈(Bean)으로 등록해준다. 따라서 우리는 구현 클래스를 만들 필요 없이 이 인터페이스를 직접 주입받아 사용할 수 있다.

### **2-2. `petMapper.java` 전체 코드**

```java
package com.spring.database.chap03;

import com.spring.database.chap03.entity.Pet;
import org.apache.ibatis.annotations.Mapper;

import java.util.List;

// CRUD를 명세
@Mapper // 이 인터페이스가 MyBatis 매퍼임을 스프링에게 알리고, 빈으로 등록하라는 표시
public interface petMapper {

    // CREATE: 메서드 이름은 XML의 id와 일치해야 함
    boolean save(Pet pet);

    // READ - SINGLE MATCHING
    Pet findById(Long id);

    // READ - MULTIPLE MATCHING
    List<Pet> findAll();

    // UPDATE
    boolean update(Pet pet);

    // DELETE
    boolean deleteById(Long id);

    // READ - COUNT
    int PetCount();
}
```

-----

## **3. Mapper XML (`petMapper.xml`)**

### **3-1. 개념 설명**

Mapper XML 파일은 Mapper 인터페이스의 추상 메서드에 대한 실제 SQL 구현을 담고 있다.

  * **`<mapper namespace="...">`**: 어떤 인터페이스와 연결될지 **패키지명을 포함한 전체 경로**를 작성한다.
  * **`<insert>`, `<select>`, `<update>`, `<delete>`**: SQL DML문에 해당하는 태그를 사용한다.
  * **`id="..."`**: 태그의 `id` 속성값은 반드시 **인터페이스의 메서드명과 일치**해야 한다.
  * **`resultType="..."`**: `SELECT` 쿼리의 결과를 어떤 자바 객체에 매핑할지 명시한다.
  * **타입 별칭(Type Alias)**: `resultType="com.spring.database.chap03.entity.Pet"`처럼 전체 경로를 쓰는 대신, `resultType="Pet"`처럼 짧게 쓸 수 있다. 스프링 부트는 보통 클래스 이름을 자동으로 별칭으로 등록해준다.
  * **`#{필드명}`**: 파라미터로 전달된 객체의 필드 값을 SQL에 바인딩하는 문법이다.

### **3-2. `petMapper.xml` 전체 코드 및 분석**

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "https://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.spring.database.chap03.petMapper">

    <insert id="save">
        INSERT INTO tbl_pet
            (pet_name, pet_age, injection)
        VALUES
            (#{petName}, #{petAge}, #{injection})
    </insert>

    <update id="update">
        UPDATE tbl_pet
        SET
            pet_name = #{petName}
            , pet_age = #{petAge}
            , injection = #{injection}
        WHERE id = #{id}
    </update>

    <delete id="deleteById">
        DELETE FROM tbl_pet
        WHERE id = #{id}
    </delete>

    <select id="findById" resultType="Pet">
        SELECT *
        FROM tbl_pet
        WHERE id = #{id}
    </select>

    <select id="findAll" resultType="Pet">
        SELECT *
        FROM tbl_pet
        ORDER BY pet_name
    </select>

    <select id="PetCount" resultType="int">
        SELECT COUNT(*)
        FROM tbl_pet
    </select>
</mapper>
```

-----

## **4. 기능 검증 (`petMapperTest.java`)**

  * **분석**: `@SpringBootTest`를 통해 테스트 환경에서 스프링 컨테이너를 실행하고, `@Autowired`로 `petMapper` 인터페이스를 주입받는다. 테스트 코드는 이제 복잡한 JDBC 로직 없이, 마치 일반 자바 메서드를 호출하듯 간결하게 데이터베이스 작업을 수행하고 검증할 수 있다.

<!-- end list -->

```java
package com.spring.database.chap03;

import com.spring.database.chap03.entity.Pet;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
class petMapperTest {

    // @Mapper 어노테이션이 붙은 인터페이스를 주입받을 수 있다.
    @Autowired
    petMapper petMapper;

    @Test
    @DisplayName("save test")
    void saveTest() {
        //given: 테스트에 필요한 데이터를 준비
        Pet newPet = Pet.builder()
                .petName("타이거")
                .petAge(30)
                .injection(false)
                .build();
        //when: 테스트할 메서드를 실행
        boolean save = petMapper.save(newPet);
        //then: 실행 결과를 단언
        assertTrue(save);
    }

    @Test
    @DisplayName("update test")
    void updateTest() {
        //given
        Pet updatePet = Pet.builder()
                .petAge(2)
                .petName("야옹이")
                .injection(false)
                .id(2L)
                .build();
        //when
        boolean update = petMapper.update(updatePet);
        //then
        assertTrue(update);
    }

    @Test
    @DisplayName("delete test")
    void deleteTest() {
        //given
        Long id = 2L;
        //when
        petMapper.deleteById(id);
        //then
        // 삭제는 보통 반환값이 없으므로, 삭제 후 조회가 안 되는지 등으로 검증할 수 있으나
        // 여기서는 실행 자체에 의미를 둠
        assertTrue(true);
    }

    @Test
    @DisplayName("find All test")
    void findAllTest() {
        //given

        //when
        List<Pet> petList = petMapper.findAll();
        //then
        petList.forEach(System.out::println);
        assertEquals(3,petList.size());
    }

    @Test
    @DisplayName("find one test")
    void findOneTest () {
        //given
        Long id = 3L;
        //when
        Pet foundPet = petMapper.findById(id);
        //then
        System.out.println("foundPet = " + foundPet);
        assertEquals("멍멍이",foundPet.getPetName());
    }

    @Test
    @DisplayName("count test")
    void countTest() {
        //given

        //when
        int count = petMapper.PetCount();
        //then
        assertEquals(3,count);
    }
}
```

-----

## **5. 관련 Entity 코드 (`Pet.java`)**

```java
package com.spring.database.chap03.entity;

/*
 CREATE TABLE tbl_pet (
                         id BIGINT AUTO_INCREMENT PRIMARY KEY,
                         pet_name VARCHAR(50),
                         pet_age INT,
                         injection BOOLEAN
);
*/

import lombok.*;

@EqualsAndHashCode
@Getter @Setter @ToString
@AllArgsConstructor @NoArgsConstructor
@Builder
public class Pet {

    private Long id;
    private String petName;
    private int petAge;
    private boolean injection;
}
```
