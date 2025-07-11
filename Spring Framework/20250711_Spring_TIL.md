
# **스프링(Spring) 웹 개발 첫걸음: 핵심 개념 정리**

-----

## **1. 스프링 부트 프로젝트의 시작과 설정**

과거의 스프링은 복잡한 XML 설정이 필요했지만,\
스프링 부트는 이러한 설정을 자동화하여 개발자가 비즈니스 로직에만 집중할 수 있게 해준다.

### **1-1. 시작점: `@SpringBootApplication`**

모든 스프링 부트 프로젝트는 하나의 `main` 메서드에서 시작한다.

```java
// SpringWebBasic202507Application.java
@SpringBootApplication // 스프링 부트의 핵심 설정 어노테이션
public class SpringWebBasic202507Application {

    public static void main(String[] args) {
        // 이 코드가 내장 톰캣 서버를 실행시키고 스프링 컨테이너를 구동시킨다.
        SpringApplication.run(SpringWebBasic202507Application.class, args);
    }
}
```

`@SpringBootApplication` 이 어노테이션 하나가 바로 스프링 부트의 "마법"의 시작이다.\
이 안에는 세 가지 핵심 어노테이션이 포함되어 있다.

1.  `@SpringBootConfiguration`: 이 프로젝트의 설정 정보를 담는 클래스임을 명시.


2.  `@EnableAutoConfiguration`: 스프링 부트가 클래스패스에 있는 라이브러리들을 스캔하여 필요한 설정들을\
**자동으로** 구성해준다. (예: `spring-boot-starter-web`이 있으면 내장 톰캣 서버를 띄우는 등)


3.  `@ComponentScan`: `@Component` 계열의 어노테이션(`@Controller`, `@Service` 등)이 붙은 클래스들을 스캔하여 **스프링 컨테이너**에 빈(Bean)으로 등록하여 관리해준다.

### **1-2. 프로젝트의 레시피: `build.gradle`**

이 파일은 우리 프로젝트에 필요한 재료(라이브러리)들의 목록과 빌드 방식을 정의하는 레시피와 같다.

```groovy
// build.gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.4.7' // 스프링 부트 플러그인
    id 'io.spring.dependency-management' version '1.1.7' // 의존성 관리 플러그인
}
// ...
dependencies {
    // 웹 개발에 필요한 핵심 라이브러리 모음 (내장 톰캣 서버 포함)
    implementation 'org.springframework.boot:spring-boot-starter-web'
    // JSP를 View로 사용하기 위한 라이브러리
    implementation 'org.apache.tomcat.embed:tomcat-embed-jasper'
    // 반복적인 코드를 줄여주는 Lombok 라이브러리
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    // 코드 변경 시 서버를 자동으로 재시작해주는 개발용 도구
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    // ... (JSTL, 테스트 관련 라이브러리 생략)
}
```

### **1-3. 프로젝트의 설정 패널: `application.properties`**

이 파일은 애플리케이션의 세부 설정을 조정하는 곳이다.

```properties
# application.properties

# 애플리케이션의 이름을 지정
spring.application.name=spring-web-basic202507

# 내장 톰캣 서버의 포트 번호를 9000으로 변경 (기본값은 8080)
server.port=9000

# MVC 패턴에서 View 파일의 경로를 설정하는 부분
# Controller가 "bye"라는 문자열을 리턴하면,
# 앞에 /WEB-INF/views/ 를 붙이고
spring.mvc.view.prefix=/WEB-INF/views/
# 뒤에 .jsp 를 붙여서
spring.mvc.view.suffix=.jsp
# 최종적으로 /WEB-INF/views/bye.jsp 파일을 찾아 렌더링한다.
```

-----

## **2. Spring MVC의 동작 원리**

Spring MVC는 스프링 프레임워크의 웹 모듈로, 웹 애플리케이션의 역할을\
**Model, View, Controller**로 나누는 MVC 디자인 패턴을 구현한다.\
이를 통해 비즈니스 로직과 사용자 인터페이스를 분리하여 코드의 유지보수성과 확장성을 높인다.

### **2-1. MVC의 3가지 구성 요소**

| 구성요소 | 역할 | 설명 |
| :--- | :--- | :--- |
| **Model** | **데이터와 비즈니스 로직** | 애플리케이션의 데이터를 처리하고 핵심 로직을 수행한다. 데이터베이스와 상호작용하며, 뷰나 컨트롤러에 의존하지 않고 독립적으로 동작한다. |
| **View** | **사용자 인터페이스(UI)** | 사용자에게 데이터를 화면으로 보여주는 부분이다. 모델로부터 받은 데이터를 JSP, Thymeleaf 등을 사용해 렌더링한다. |
| **Controller** | **중재자, 요청 처리** | 사용자의 요청을 받아 처리하고, 필요한 데이터를 모델에서 가져와 뷰에 전달하는 연결고리 역할을 한다. |

### **2-2. Spring MVC의 동작 흐름**

클라이언트의 요청이 들어왔을 때 Spring MVC가 처리하는 과정은 다음과 같다.

1.  **사용자 요청**: 클라이언트가 URL을 통해 요청을 보낸다.


2.  **DispatcherServlet**: 스프링의 **프론트 컨트롤러**인 디스패처 서블릿이 모든 요청을 가장 먼저 받아서,\
요청에 맞는 적절한 컨트롤러로 전달한다.


3.  **Controller**: 요청을 위임받은 컨트롤러가 비즈니스 로직을 실행하고, 그 결과를 **Model** 객체에 저장한다.\
로직 처리 후, 어떤 View를 보여줄지 **View의 논리적 이름**을 반환한다. (예: `"hello"`)


4.  **View Resolver**: 컨트롤러가 반환한 뷰 이름을 보고, 실제 물리적인 뷰 파일을 찾아낸다.\
예를 들어 `application.properties`의 설정(`prefix`, `suffix`)을 조합하여\
`"hello"`를 `/WEB-INF/views/hello.jsp`로 변환한다.


5.  **View**: 최종적으로 뷰는 Model에 담긴 데이터를 사용하여 화면을 그리고,\
완성된 HTML을 사용자에게 응답한다.

### **2-3. Spring MVC의 주요 장점**

* 모듈화된 구조로 비즈니스 로직과 UI 로직을 명확하게 분리할 수 있다.
* JSP, Thymeleaf 등 다양한 뷰 기술과 유연하게 통합된다.
* 어노테이션 기반의 설정을 통해 개발 생산성을 높일 수 있다.

-----

## **3. 컨트롤러와 요청 처리**

클라이언트의 모든 웹 요청은 스프링의 \*\*디스패처 서블릿(Dispatcher Servlet)\*\*이 가장 먼저 받는다. 그리고 `@RequestMapping` 등의 정보를 보고, 어떤 컨트롤러의 어떤 메서드가 이 요청을 처리해야 할지 결정하여 작업을 위임한다.

### **3-1. 컨트롤러의 종류**

| 어노테이션 | 역할 | 설명 |
| :--- | :--- | :--- |
| **`@Controller`** | **View 반환** | 주로 JSP 같은 뷰 템플릿 파일을 찾아 렌더링한 결과를 반환한다. 메서드가 문자열을 반환하면, `application.properties`의 설정에 따라 해당 경로의 파일을 찾는다. |
| **`@RestController`** | **데이터(JSON) 반환** | `@Controller`와 `@ResponseBody`가 합쳐진 어노테이션. 메서드의 반환 값을 View로 해석하지 않고, JSON 형태의 데이터로 변환하여 클라이언트에게 직접 응답한다. API 서버 개발에 사용된다. |

### **3-2. 요청 매핑 어노테이션 (Request Mapping Annotations)**

URL 요청을 특정 컨트롤러 메서드와 연결(매핑)하는 역할을 한다.

| 어노테이션 | 설명 |
| :--- | :--- |
| **`@RequestMapping`** | 가장 기본적인 매핑 어노테이션. `value` 속성으로 URL을, `method` 속성으로 HTTP 메서드(GET, POST 등)를 지정할 수 있다. 클래스 레벨에 붙이면 공통 URL을 처리할 수 있다. |
| **`@GetMapping`** | `@RequestMapping(method = RequestMethod.GET)`의 축약형. GET 요청만 처리한다. |
| **`@PostMapping`** | `@RequestMapping(method = RequestMethod.POST)`의 축약형. POST 요청만 처리한다. |
| **`@RequestParam`** | URL의 쿼리 파라미터(`?key=value`)를 메서드의 파라미터 변수에 바인딩해준다. `required`, `defaultValue` 등의 속성으로 세부 설정을 할 수 있다. |

-----

### **3-3. 요청 파라미터(Query String)란?**

* **쿼리 스트링(Query String)** 은 클라이언트가 GET 방식 요청을 보낼 때\
서버에 데이터를 함께 전달하기 위해 사용하는 URL의 일부이다.\
주로 데이터를 필터링, 검색, 정렬하는 등의 목적으로 사용된다.


* **구조**: URL의 경로 뒤에 `?`를 붙여 시작하며, `key=value` 형태의 쌍으로 구성된다.\
여러 개의 파라미터는 `&` 기호로 연결한다다.
  ```
  /products?id=2&sort=price&category=elec
  ```
* **분석**: 위 URL은 "id가 2이고, 정렬 기준은 price이며\
카테고리는 elec인 상품을 조회"하려는 의도를 담고 있다.

### **3-4. 요청 파라미터 처리 (`@RequestParam`)**

`@RequestParam` 어노테이션은 컨트롤러 메서드에서\
 이 쿼리 스트링의 값을 쉽게 읽어올 수 있도록 도와준다.

| 어노테이션 | 설명                                                                                                    |
| :--- |:------------------------------------------------------------------------------------------------------|
| **`@RequestParam`** | URL의 쿼리 파라미터(`?key=value`)를 메서드의 파라미터 변수에 바인딩해준다. `required`, `defaultValue` 등의 속성으로 세부 설정을 할 수 있습니다. |

---

## **4. 코드 분석: 요청부터 응답까지**

### **`HelloController.java` - View와 JSON을 함께 처리하는 컨트롤러**

* **View 반환**: `/chap01/bye` 요청이 오면 `bye()` 메서드가 실행된다.


    * 이 메서드는 `"bye"`라는 문자열을 반환한다.
    * 디스패처 서블릿은 `application.properties`의 `prefix`와 `suffix` 설정을 참고하여,\
      최종 경로인 `/WEB-INF/views/bye.jsp`를 찾아 **포워딩**한다.


* **JSON 데이터 반환**: `/chap01/hello` 요청이 오면 `hello()` 메서드가 실행된다.


    * `@ResponseBody` 어노테이션 때문에, 반환값인 `Map` 객체를 JSON 형태로 변환하여 응답 본문에 직접 써준다.

<!-- end list -->

```java
// HelloController.java
@Controller
public class HelloController {

    // http://localhost:9000/chap01/hello
    @RequestMapping("/chap01/hello")
    @ResponseBody // 이 메서드는 View를 찾지 말고, 반환값을 JSON으로 만들어 응답해라!
    public Map<String,Object> hello() {
        return Map.of("name", "김철수", "age", 50);
    }

    // http://localhost:9000/chap01/bye
    @RequestMapping("/chap01/bye")
    public String bye() {
        // "bye"를 리턴하면, 스프링이 /WEB-INF/views/bye.jsp 를 찾아서 열어준다.
        return "bye";
    }
}
```

### **`ProductController.java` - 요청 파라미터 처리**

`/products?id=2&price=5000` 과 같은 요청에서 쿼리 파라미터를 읽어오는 예제

```java
// ProductController.java
@RestController
@RequestMapping("/products")
public class ProductController {
    // ...
    @GetMapping
    public Product getProduct(
            // ?id=xxx 값을 long id 변수에 넣어줘.
            @RequestParam("id") long id,
            // ?price=xxx 값을 int price 변수에 넣어줘. 필수는 아니고, 없으면 1000을 기본값으로 해줘.
            @RequestParam(value = "price", required = false, defaultValue = "1000") int price
    ) {
        System.out.println("id = " + id);
        System.out.println("price = " + price);
        return productMap.get(id);
    }
}
```

`@RequestParam` 덕분에 과거 서블릿처럼 `request.getParameter()`를 쓰고\
타입 변환을 하던 번거로운 작업이 사라졌다.

### **유용한 도구: Lombok**

`Product.java` 클래스에서는 `@Getter`, `@Setter` 같은 롬복(Lombok) 어노테이션을 사용했다.\
이 어노테이션들은 컴파일 과정에서 자동으로 `getter`, `setter`, `toString` 등의 메서드를 생성해주어,\
자바 모델 클래스의 코드를 매우 깔끔하게 유지할 수 있게 도와준다.