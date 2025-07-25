
# **웹 개발 핵심 개념 상세 정리**

## **1. 네트워크 통신의 기초**

인터넷 통신은 클라이언트와 서버가 **IP 주소**를 통해 서로를 식별하고, **패킷**이라는 데이터 조각을 주고받는 방식으로 이루어진다. 하지만 IP 프로토콜만으로는 안정적인 통신에 한계가 있어 TCP 프로토콜이 이를 보완한다.

### **IP 프로토콜의 3가지 한계**

1.  **비연결성**: 패킷을 받을 대상이 없거나 서비스 불능 상태여도 일단 패킷을 전송한다.
2.  **비신뢰성**: 패킷이 중간에 사라지거나, 순서가 뒤바뀌어 전달될 수 있다.
3.  **프로그램 구분 불가**: 같은 컴퓨터의 여러 앱에서 동시에 데이터를 보내도, 받는 쪽에서 어떤 앱이 보낸 것인지 구분할 방법이 없다.

### **TCP: 신뢰성 있는 통신을 위한 약속**

TCP(Transmission Control Protocol)는 IP의 한계를 극복하고 신뢰성 있는 데이터 전송을 보장한다.

* **3-way-handshake**: 데이터를 보내기 전, 클라이언트와 서버가 총 3번의 통신(`SYN` → `SYN/ACK` → `ACK`)으로 서로의 연결 상태를 확인하여 **연결성**과 **신뢰성**을 확보한다.
* **PORT 번호**: 하나의 IP 주소 내에서 실행되는 여러 애플리케이션을 구분하기 위한 번호다. TCP 데이터에는 이 포트 정보가 포함되어, 특정 애플리케이션 간의 정확한 통신이 가능해진다.
* **데이터 순서 보장**: TCP는 전송된 데이터 패킷의 순서를 보장하여 데이터가 올바른 순서로 조립되도록 한다.

### **DNS: IP 주소를 위한 전화번호부**

DNS(Domain Name System)는 `172.217.175.68`과 같이 기억하기 어려운 IP 주소를 `google.com`처럼 사람이 기억하기 쉬운 도메인 이름으로 변환해주는 시스템이다.

---

## **2. 웹 통신과 HTTP**

웹브라우저(클라이언트)와 웹 서버는 **HTTP(HyperText Transfer Protocol)**라는 규칙을 통해 통신한다. 이때 특정 자원을 식별하기 위해 **URI**를 사용한다.

### **URI와 URL**

* **URI (Uniform Resource Identifier)**: 인터넷의 자원을 식별하는 통일된 방법.
* **URL (Uniform Resource Locator)**: 자원의 위치를 나타내며, 우리가 흔히 쓰는 웹 주소다. URI의 한 종류이며 사실상 URI와 동일한 의미로 사용된다.
* **URL의 구조**: `scheme://host:port/path?query#fragment`
    * `scheme`: 통신 프로토콜 (`http`, `https`).
    * `host`: 서버의 도메인명이나 IP 주소.
    * `port`: 서버 애플리케이션 포트 (기본값: http 80, https 443).
    * `path`: 서버 내 자원의 경로.
    * `query`: `?key=value` 형태로 서버에 전달하는 데이터.

### **HTTP의 핵심 특징**

* **클라이언트-서버 구조**: 클라이언트가 요청하고 서버가 응답하는 명확한 역할 분리.
* **무상태(Stateless)**: 서버가 클라이언트의 이전 요청 상태를 저장하지 않는다.
    * **장점**: 상태에 얽매이지 않으므로 서버를 무한히 증설할 수 있어 확장성이 좋다.
    * **단점**: 로그인과 같이 상태 유지가 필요한 경우, 쿠키, 세션, 토큰 등의 별도 기술을 사용해야 한다.
* **비연결성(Connectionless)**: 서버가 요청에 응답한 후 바로 연결을 끊어버려, 많은 사용자의 요청을 효율적으로 처리하고 서버 자원을 아낄 수 있다.


---

### **HTTP 무상태(Stateless) 핵심 요약**

#### **정의**

* 서버가 클라이언트의 이전 요청을 **전혀 기억하지 못하는** 특성입니다.
* 모든 요청은 각각 완전히 **독립적인, 새로운 요청**으로 취급됩니다.

#### **단점 (문제점)**

* 서버가 이전 요청을 기억하지 못하므로, **연속적인 작업**(예: 로그인 후 글쓰기)이 불가능합니다.
* **노트북 가게 예시**: 점원이 계속 바뀌어서 매번 "노트북 사러 왔어요"라고 처음부터 다시 말해야 하는 것과 같습니다.

#### **장점 (왜 사용하는가?)**

* **서버 확장성**이 매우 좋습니다. 클라이언트의 상태를 기억할 필요가 없으므로, 어떤 서버든 요청을 받아 처리할 수 있어 서버 수를 쉽게 늘릴 수 있습니다.

#### **해결책 (어떻게 상태를 유지하는가?)**

* **쿠키와 세션/토큰** 기술을 사용합니다.
  1.  **로그인 성공**: 서버가 클라이언트에게 "인증 티켓"(토큰 등)을 발급합니다.
  2.  **요청 시 티켓 제출**: 클라이언트는 이후 모든 요청에 이 티켓을 함께 실어 보냅니다.
  3.  **서버에서 식별**: 서버는 티켓을 보고 클라이언트를 식별하여, 상태가 유지되는 것처럼 동작합니다.

---

### **HTTP 헤더와 상태 코드**

* **HTTP 헤더**: 클라이언트와 서버가 통신할 때 필요한 부가 정보를 담는다. 데이터 형식(`Content-Type`), 인증 정보(`Authorization`), 캐싱 정책(`Cache-Control`) 등이 헤더에 포함된다.
* **HTTP 상태 코드**: 요청 처리 결과를 나타내는 세 자리 숫자.
    * `2xx (성공)`: `200 OK`(요청 성공), `201 Created`(리소스 생성 성공).
    * `3xx (리다이렉션)`: `301 Moved Permanently`(영구 이동).
    * `4xx (클라이언트 오류)`: `400 Bad Request`(요청 형식 오류), `401 Unauthorized`(인증 필요), `403 Forbidden`(접근 권한 없음), `404 Not Found`(리소스 없음).
    * `5xx (서버 오류)`: `500 Internal Server Error`(서버 내부 문제).

---

## **3. 웹 서버 아키텍처**

### **웹 서버 vs. WAS (Web Application Server)**

* **웹 서버 (Web Server)**: **정적 컨텐츠**(HTML, CSS, 이미지)를 제공하는 데 특화된 서버. (예: Apache, Nginx)
* **WAS (Web Application Server)**: 데이터베이스 조회 등 비즈니스 로직을 수행하여 **동적 컨텐츠**를 생성하는 데 특화된 서버. 서블릿 컨테이너 역할도 수행한다. (예: Tomcat, JBoss)

### **웹 서버와 WAS를 분리하는 이유**

역할을 분담하여 서버 부하를 줄이고, 시스템의 안정성과 효율성을 높이기 위함이다.
1.  **확장성**: 정적 컨텐츠 요청이 많으면 웹 서버만, 동적 컨텐츠 요청이 많으면 WAS만 증설하면 되어 효율적이다.
2.  **보안성**: 웹 서버를 앞단에 두어 중요한 비즈니스 로직이 담긴 WAS를 외부 공격으로부터 보호할 수 있다.
3.  **유지보수성**: 역할이 분리되어 있어 각 서버의 유지보수 및 업그레이드가 용이하다.

### **CSR vs. SSR**

* **CSR (Client Side Rendering)**: 브라우저가 대부분의 UI 렌더링을 처리한다. 최초 로딩은 느리지만 이후 페이지 전환이 빠르다. 주로 **SPA(Single Page Application)** 방식에 사용된다. SEO에 취약하다.
* **SSR (Server Side Rendering)**: 서버에서 완전히 렌더링된 HTML 페이지를 제공한다. 초기 로딩이 빠르고 SEO에 유리하다. 주로 **MPA(Multiple Page Application)** 방식에 사용된다.

---

## **4. 자바 웹 기술의 발전**

### **Servlet과 JSP**

* **Servlet**: 자바 코드를 기반으로 웹 페이지를 만드는 기술. 자바 코드 안에서 HTML을 문자열로 작성해야 해서 매우 번거롭다.
* **JSP (Java Server Pages)**: HTML 문서 안에 자바 코드를 삽입하여 동적 페이지를 만드는 기술. 서블릿보다 화면 개발이 편리하다.

### **MVC 아키텍처**

서블릿과 JSP만으로는 비즈니스 로직과 화면 코드가 뒤섞여 유지보수가 어려웠다. 이를 해결하기 위해 **MVC 아키텍처**가 등장했다.
* **Model**: 데이터와 비즈니스 로직을 처리.
* **View**: 사용자에게 보여줄 화면(UI)을 담당. (주로 JSP)
* **Controller**: 사용자의 요청을 받아 Model과 View를 연결하는 역할. (주로 Servlet)

### **Spring Framework**

Struts 같은 초기 프레임워크를 거쳐, **스프링(Spring)** 이 등장하며 복잡한 MVC 패턴 구현을 훨씬 쉽게 만들어주었다. DI, AOP와 같은 기능을 통해 개발자들이 비즈니스 로직에만 집중할 수 있게 도와주는 현대적인 자바 웹 프레임워크다.

---

## **5. 빌드 도구**

### **빌드 도구의 역할**

프로젝트 생성부터 컴파일, 테스트, 라이브러리 관리, 배포까지 전 과정을 자동화하여 개발 생산성을 높여주는 도구다.
1.  **의존성 관리**: 프로젝트에 필요한 라이브러리(jar)를 자동으로 다운로드하고 관리해준다.
2.  **빌드 자동화**: 컴파일, 테스트, 패키징 등의 작업을 자동으로 수행한다.

### **Maven vs. Gradle**

* **Maven**: XML (`pom.xml`) 기반의 빌드 도구. 설정이 비교적 간단하고, [중앙 리포지토리](https://search.maven.org/)를 통해 방대한 라이브러리를 지원한다.
* **Gradle**: Groovy/Kotlin DSL (`build.gradle`) 기반의 빌드 도구. Maven보다 유연성이 높고, 빌드 캐시 기능 덕분에 빌드 속도가 더 빠르다.
