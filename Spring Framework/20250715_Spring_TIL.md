
# **Thymeleaf 템플릿 엔진 학습 정리**

오늘은 JSP를 대체하는 현대적인 서버 사이드 템플릿 엔진인 \*\*타임리프(Thymeleaf)\*\*에 대해 학습했다.\
스프링 부트와의 강력한 통합 기능을 바탕으로, 컨트롤러가 전달한 데이터를 어떻게 HTML 페이지에 렌더링하는지 그 원리를 익혔다.

-----

## **1. Thymeleaf의 주요 특징**

타임리프는 JSP와 마찬가지로 서버에서 동적으로 HTML 페이지를 생성하는 기술이지만, 몇 가지 중요한 차이점이 있다.

* **내추럴 템플릿 (Natural Templates)**: 타임리프 파일(`.html`)은 그 자체로 완전한 HTML 파일이다\
따라서 서버를 실행하지 않고 파일을 웹 브라우저에서 직접 열어도 깨지지 않고 원본 HTML 구조를 확인할 수 있다.\
이는 웹 디자이너와의 협업을 매우 용이하게 만든다.


* **스프링 부트와의 강력한 통합**: 스프링 부트는 타임리프에 대한 자동 설정(Auto-Configuration)을 지원하므로,\
별도의 `ViewResolver` 설정 없이 `build.gradle`에 의존성만 추가하면 바로 사용할 수 있다.


* **표준 HTML 속성 사용**: 모든 타임리프 문법은 `th:*` 형태의 HTML 속성으로 작성되므로,\
기존 HTML의 구조를 해치지 않는다.

-----

## **2. Controller - 데이터 전달 방식**

컨트롤러가 뷰에 데이터를 전달하는 방식은 JSP를 사용할 때와 **완전히 동일하다.**\
`@Controller` 어노테이션을 사용하며, `Model` 객체에 데이터를 담아 뷰의 논리적 이름을 반환한다.

### **`ThymeLeafController.java` 전체 코드 및 분석**

```java
package com.spring.basic.chap4_2;

import org.springframework.ui.Model; // JSP 때와 동일한 Model 객체 사용

import java.util.List;

@Controller
@RequestMapping("/chap4-2")
public class ThymeLeafController {

    @GetMapping("/hobby-page")
    public String hobbyPage(Model model) {
        // 1. Model 객체에 데이터 담기 (JSP와 동일)
        model.addAttribute("username", "또치");
        model.addAttribute("hobbies", List.of("동생괴롭히기", "폭식하기", "쐬질하기"));
        
        // 2. View의 논리적 이름 반환 (JSP와 동일)
        // 스프링 부트 타임리프 기본 설정:
        // prefix: src/main/resources/templates/
        // suffix: .html
        // 최종적으로 /templates/hobby.html 파일을 찾아 렌더링함.
        return "hobby";
    }
}
```

* **분석**: 위 코드에서 볼 수 있듯이, 뷰 기술이 JSP에서 타임리프로 바뀌어도 컨트롤러의 코드는 전혀 변경되지 않는다.\
이것은 스프링 MVC가 뷰 기술에 비종속적으로 설계되었음을 보여주는 좋은 예다.

-----

## **3. View - Thymeleaf 문법**

JSP가 `<%...%>`나 `<c:forEach>` 같은 태그를 사용했다면, \
타임리프는 `th:*` 속성을 사용하여 데이터를 표현한다.

### **`hobby.html` 전체 코드 및 분석**

```html
<!DOCTYPE html>
<html lang="ko" xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="UTF-8">
  <title>Title</title>
</head>
<body>
  <h1>타임리프 hobby페이지입니다.</h1>
  
  <h2 th:text="${username}">이 부분은 서버 실행 시 '또치'로 교체됨</h2>
  
  <ul>
    <li th:each="hobby : ${hobbies}" th:text="${hobby}">
        이 내용은 반복 횟수만큼 hobby 변수의 값으로 교체됨
    </li>
  </ul>
</body>
</html>
```

* **핵심 문법**:
    * **`th:text="${...}"`**: 태그 내부의 텍스트를 지정된 모델 데이터의 값으로 변경한다.\
  JSP의 `${...}` 표현식과 유사하다.
    * **`th:each="변수명 : ${컬렉션}"`**: JSTL의 `<c:forEach>`처럼 컬렉션 데이터를 반복 처리한다.\
  해당 태그 자체가 반복 생성된다.

-----

## **4. JSP vs. Thymeleaf 비교**

| 구분 | JSP | Thymeleaf |
| :--- | :--- | :--- |
| **파일 형식** | `.jsp` (HTML + Java 코드) | `.html` (순수 HTML) |
| **독립 실행** | **불가능**. 서버 없이 브라우저에서 열면 코드가 그대로 보인다. | **가능**. 서버 없이도 원본 HTML 구조를 확인할 수 있다. |
| **주요 문법** | `<% ... %>`, `<%= ... %>`, `<c:forEach>` | `th:*` 속성 (e.g., `th:text`, `th:each`) |
| **스프링 부트** | 수동 의존성 추가 및 설정 필요. | **공식 지원 및 추천**. `spring-boot-starter-thymeleaf` 의존성 하나로 자동 설정 완료. |