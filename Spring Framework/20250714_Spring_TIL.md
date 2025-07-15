
# **Spring MVC와 JSP 연동: 서버 사이드 렌더링(SSR)**

오늘은 스프링 부트 환경에서 \*\*JSP(JavaServer Pages)\*\*를 뷰(View)로 사용하여 서버 측에서\
완전한 HTML 페이지를 만들어 클라이언트에게 응답하는 **서버 사이드 렌더링(Server-Side Rendering)** 방식을 학습했다.\
이 과정에서 \*\*`@Controller`\*\*와 **`Model` 객체**의 역할이 핵심적이다.

-----

## **1. `@Controller` vs. `@RestController`**

먼저 두 어노테이션의 차이를 명확히 해야 한다.

* `@RestController`: 메서드의 반환값을 JSON이나 텍스트 같은 **데이터 자체**로 응답한다. API 서버 개발에 사용된다.


* `@Controller`: 메서드가 반환하는 **문자열을 뷰(View)의 이름으로** 해석한다. 스프링의 `ViewResolver`가 이 이름을 바탕으로\
실제 뷰 파일(e.g., `.jsp`)을 찾아 포워딩한다. 오늘 학습한 서버 사이드 렌더링에 사용된다.

-----

## **2. Controller에서 View로 데이터 전달하기**

컨트롤러는 비즈니스 로직을 처리한 후, 그 결과를 화면에 보여주기 위해 뷰(JSP)에게 데이터를 전달해야 한다.\
이때 사용하는 것이 바로 `Model` 객체다.

### **2-1. `Model` 객체의 역할**

`Model`은 컨트롤러에서 생성한 데이터를 JSP로 전달하기 위한 **운반용 상자**와 같은 역할을 한다.\
컨트롤러 메서드의 파라미터로 `Model`을 선언하면, 스프링이 자동으로 `Model` 객체를 생성하여 주입해준다.

* `model.addAttribute("key", value)`: `"key"`라는 이름표를 붙여 `value` 데이터를 상자에 담는다.

### **2-2. `PetController4_1.java` 전체 코드 및 분석**

```java
package com.spring.basic.chap4_1.controller;

// 이 클래스는 View를 찾아 렌더링하는 역할을 함
@Controller
@RequestMapping("/chap4-1")
public class PetController4_1 {

    private static long nextNo = 1;

    // DB 대신 사용할 임시 펫 리스트
    private List<Pet> petList = new ArrayList<>();

    // 컨트롤러 생성 시 초기 데이터 추가
    public PetController4_1() {
        petList.add(Pet.builder()
                        .id(nextNo++)
                        .age(3)
                        .name("뽀삐")
                        .kind("불독")
                        .injection(true)
                .build());
        petList.add(Pet.builder()
                .id(nextNo++)
                .age(4)
                .name("초코")
                .kind("도베르만")
                .injection(false)
                .build());
        petList.add(Pet.builder()
                .id(nextNo++)
                .age(3)
                .name("나비")
                .kind("벵갈호랑이")
                .injection(false)
                .build());
    }

    // pet.jsp를 열여줘 (뷰 포워딩) - 페이지 라우팅
    @GetMapping("/pet-page")
    public String petPage(Model model) { // Spring이 Model 객체를 주입해 줌
        
        // 서버사이드 렌더링을 위해 JSP파일에게 리스트를 전달
        // "petList"라는 이름표(key)로 petList(value)를 모델에 담는다.
        model.addAttribute("petList",petList);
        
        // "chap4-1/pet" 라는 뷰 이름을 리턴
        // ViewResolver가 이 이름을 조합하여 실제 JSP 경로를 찾아 포워딩
        // -> /WEB-INF/views/chap4-1/pet.jsp
        return "chap4-1/pet";
    }
}
```

-----

## **3. View(JSP)에서 데이터 렌더링하기**

컨트롤러로부터 전달받은 `Model`의 데이터를 JSP에서는 **EL(Expression Language)** 과\
**JSTL(JSP Standard Tag Library)** 을 사용하여 화면에 그린다.

### **3-1. `pet.jsp` 전체 코드**

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Web Study</title>
</head>
<body>

    <h1>애완동물 페이지입니다.</h1>
  <%--  애완동물 목록을 서버 사이드 랜더링 --%>
    <ul>
        <%-- JSTL의 forEach 태그를 사용하여 Model에 담겨온 petList를 순회 --%>
        <%-- items 속성에는 Model의 key값("petList")을 EL로 작성 --%>
        <c:forEach var="pet" items ="${petList}">
        <li>
            <%-- EL을 사용하여 각 pet 객체의 프로퍼티(필드) 값을 출력 --%>
            이름: ${pet.name},종: ${pet.kind},
            나이: ${pet.age}살,접종여부: ${pet.injection}
        </li>
        </c:forEach>
    </ul>

</body>
</html>
```

### **3-2. 코드 분석 및 전체 동작 흐름**

1.  **클라이언트 요청**: 브라우저에서 `http://localhost:포트/chap4-1/pet-page`를 요청한다.


2.  **`DispatcherServlet`**: 요청을 받아 `@GetMapping("/pet-page")`가 있는\
`PetController4_1`의 `petPage` 메서드를 찾아 실행한다.


3.  **컨트롤러 처리**: `petPage` 메서드는 `petList` 데이터를 생성하고,\
`model.addAttribute("petList", petList)` 코드를 통해 이 데이터를 `Model` 객체에 담는다.


4.  **뷰 이름 반환**: 컨트롤러는 뷰의 논리적 이름인 `"chap4-1/pet"`을 반환한다.


5.  **`ViewResolver` 동작**: 스프링의 `ViewResolver`는 `application.properties`에 설정된\
`prefix`와 `suffix`를 조합하여, 최종 물리적 경로인 `/WEB-INF/views/chap4-1/pet.jsp`를 찾아낸다.


6.  **데이터 전달 및 렌더링**: `Model`에 담겨 있던\
`petList` 데이터가 `pet.jsp`로 전달된다.


7.  **JSP 실행**: `pet.jsp`는 서버 위에서 실행된다.\
`<c:forEach>`와 `${...}`를 통해 `petList`의 데이터를 포함한 **완전한 HTML 코드**를 생성한다.


8.  **최종 응답**: 완성된 HTML 코드가 클라이언트의 브라우저로 전송되어 사용자에게 최종 화면이 보여진다.