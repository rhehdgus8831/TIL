

# **서블릿(Servlet) 기반 웹 애플리케이션 분석**

오늘은 초기 자바 웹 기술인 서블릿(Servlet)을 사용하여 댄서 등록 및 조회 웹 애플리케이션을 구현했다. 이 프로젝트를 통해 HTTP 요청과 응답을 처리하고, 서버 측에서 동적으로 HTML을 생성하는 서블릿의 핵심 동작 원리를 학습했다.

-----

## **1. 애플리케이션의 전체 흐름**

이 애플리케이션은 사용자의 요청(Request)에 따라 각기 다른 서블릿이 응답(Response)하며 동작한다.

1.  **등록 페이지 요청**: 사용자가 `/chap02/dancer/register` URL로 접속한다.
    - `DancerRegisterServlet`이 요청을 받아, 댄서 정보를 입력할 수 있는 HTML `<form>`을 생성하여 클라이언트에게 전송한다.
2.  **등록 데이터 전송**: 사용자가 폼에 데이터를 입력하고 '등록' 버튼을 클릭한다.
    - 브라우저는 폼 데이터를 `action` 속성에 지정된 `/chap02/dancer/process` URL로 전송한다.
3.  **등록 처리**: `DancerProcessServlet`이 이 요청을 받아 폼 데이터를 읽어온다.
    - 읽어온 데이터로 `Dancer` 객체를 생성한다.
    - 생성된 객체를 `DancerRepository`에 저장한다.
    - "등록 성공" 메시지와 함께 댄서 목록 페이지로 이동할 수 있는 링크를 담은 HTML을 응답한다.
4.  **목록 페이지 요청**: 사용자가 등록 성공 페이지의 링크를 클릭하여 `/chap02/dancer/show-list` URL로 접속한다.
    - `DancerListServlet`이 요청을 받아, `DancerRepository`에 저장된 모든 댄서 목록을 조회한다.
    - 조회된 목록을 기반으로 동적으로 `<li>` 태그를 생성하여 전체 댄서 목록이 담긴 HTML 페이지를 응답한다.

-----

## **2. 핵심 구성 요소 분석**

### **2-1. 서블릿(Servlet)의 역할과 동작**

서블릿은 HTTP 통신을 쉽게 처리해주는 자바 클래스다. `HttpServlet`을 상속받아 만들며, 특정 URL 요청을 처리하는 역할을 한다.

* **URL 매핑 (`@WebServlet`)**: 클래스 상단에 `@WebServlet("/경로")` 어노테이션을 붙여 어떤 URL 요청을 이 서블릿이 처리할지 지정한다.
* **요청 처리 (`HttpServletRequest`)**: 클라이언트가 보낸 요청에 대한 모든 정보를 `req` 객체를 통해 얻을 수 있다.
    - `req.getParameter("파라미터명")`: URL 쿼리 스트링이나 폼 데이터를 읽어온다.
* **응답 생성 (`HttpServletResponse`)**: `resp` 객체를 통해 클라이언트에게 보낼 응답을 생성한다.
    1.  `resp.setContentType("text/html")`: 응답 데이터가 HTML 문서임을 명시한다.
    2.  `resp.setCharacterEncoding("UTF-8")`: 한글 등 비영어권 문자가 깨지지 않도록 인코딩을 설정한다.
    3.  `PrintWriter w = resp.getWriter()`: HTML 코드를 작성할 수 있는 '펜'(`PrintWriter` 객체)을 얻는다.
    4.  `w.write("...")`: `write()` 메서드를 사용하여 `<!DOCTYPE html>`부터 `</html>`까지 HTML 문자열을 직접 한 줄 한 줄 작성하여 동적인 웹 페이지를 만든다.

### **2-2. 데이터 모델과 저장소**

* **데이터 모델 (`Dancer.java`, `Genre.java`, `DanceLevel.java`)**: 애플리케이션에서 사용할 데이터의 구조를 정의한 클래스들이다. `Dancer`는 댄서 한 명의 정보를 담는 객체다.
* **데이터 저장소 (`DancerRepository.java`)**: 댄서 객체들을 관리하는 클래스. 실제 데이터베이스 대신 `static` `ArrayList`를 사용하여 프로그램이 실행되는 동안 댄서 목록을 메모리에 저장하고 관리하는 역할을 한다. `addDancer`, `getDancerList` 같은 메서드를 제공한다.

### **2-3. 내장 톰캣 서버 (`JspStarterMain.java`)**

이 프로젝트는 외부의 톰캣(WAS) 서버를 설치하지 않고도 실행할 수 있다. `JspStarterMain` 클래스가 자바 코드만으로 내장 톰캣 서버를 구동시켜, 개발 환경을 간소화하고 애플리케이션을 쉽게 실행할 수 있도록 돕는다.

-----

## **3. 주요 서블릿 기능 상세 분석**

### **`DancerRegisterServlet.java`**

- **역할**: 댄서 등록을 위한 HTML 폼 페이지 제공.
- **동작**: 오직 `PrintWriter`를 사용하여 정적인 HTML `<form>` 태그를 클라이언트에게 그려주는 역할만 수행한다. 이 폼의 제출(submit) 목적지는 `/chap02/dancer/process`로 지정되어 있다.

**주요 코드**

```java
// DancerRegisterServlet.java
@Override
protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    // 새로운 등롤을 위한 form태그가 나와야함.

    resp.setContentType("text/html");
    resp.setCharacterEncoding("utf-8");

    PrintWriter w = resp.getWriter();

    w.write("<!DOCTYPE html>\n");
    w.write("<html>\n");
    w.write("<head>\n");
    // ... 스타일 생략 ...
    w.write("</head>\n");
    w.write("<body>\n");
    w.write("<form action=\"/chap02/dancer/process\" method=\"get\" id=\"reg-form\">");
    w.write("<label># 이름 : <input type=\"text\" name=\"name\"></label>");
    w.write("<label># 크루이름 : <input type=\"text\" name=\"crewName\"></label>");
    w.write("<label># 레벨 :<input type=\"radio\" name=\"danceLevel\" value=\"PROFESSIONAL\"> 프로 <input type=\"radio\" name=\"danceLevel\" value=\"AMATEUR\"> 아마추어 <input type=\"radio\" name=\"danceLevel\" value=\"BEGINNER\"> 초보자 </label>");
    w.write("<label># 장르 :<input type=\"checkbox\" name=\"genres\" value=\"HIPHOP\"> 힙합 <input type=\"checkbox\" name=\"genres\" value=\"STREET\"> 스트릿 <input type=\"checkbox\" name=\"genres\" value=\"KPOP\"> 케이팝 </label>");
    w.write("<label><button id=\"reg-btn\" type=\"submit\">등록</button></label>");
    w.write("</form>");
    w.write("</body>\n");
    w.write("</html>");
}
```

### **`DancerProcessServlet.java`**

- **역할**: 폼에서 전송된 데이터 처리 및 저장.
- **동작**: `req.getParameter()`와 `req.getParameterValues()`를 사용해 `name`, `crewName` 등의 폼 데이터를 읽는다. 읽어온 데이터로 새로운 `Dancer` 객체를 생성한 후 `DancerRepository.addDancer()`를 호출하여 리스트에 추가한다. 처리 후 간단한 성공 페이지를 응답한다.

**주요 코드**

```java
// DancerProcessServlet.java
@Override
protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

    // form에서 입력된 데이터를 읽어서 실제로 댄서 목록에 추가해줘야함.
    String name = req.getParameter("name");
    String crewName = req.getParameter("crewName");
    String danceLevel = req.getParameter("danceLevel");
    String[] genres = req.getParameterValues("genres");

    List<Genre> genreList = new ArrayList<>();
    for (String g : genres) {
        genreList.add(Genre.valueOf(g));
    }

    Dancer dancer = new Dancer(
            name, crewName, DanceLevel.valueOf(danceLevel), genreList
    );

    DancerRepository.addDancer(dancer);
    
    // 응답 html 생성 ...
}
```

### **`DancerListServlet.java`**

- **역할**: 저장된 전체 댄서 목록 조회 페이지 제공.
- **동작**: `DancerRepository.getDancerList()`를 통해 모든 댄서 목록을 가져온다. `for` 반복문을 사용하여 리스트의 각 `Dancer` 객체를 순회하며, 각 댄서의 정보를 담은 `<li>` 태그를 동적으로 생성하여 HTML 목록을 만든다. 이것이 바로 서블릿을 이용한 \*\*서버 사이드 렌더링(SSR)\*\*의 가장 기본적인 예시다.

**주요 코드**

```java
// DancerListServlet.java
@Override
protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

    // 댄서 목록 불러오기
    List<Dancer> dancerList = DancerRepository.getDancerList();

    // 결과 출력
    resp.setContentType("text/html");
    resp.setCharacterEncoding("UTF-8");

    PrintWriter w = resp.getWriter();

    w.write("<!DOCTYPE html>\n");
    w.write("<html>\n");
    w.write("<head>\n");
    w.write("</head>\n");
    w.write("<body>\n");
    w.write("<ul>");

    for (Dancer foundDancer : dancerList) {
        w.write(String.format("<li># 이름 : %s, 크루명: %s, 수준: %s, 장르: %s</li>\n"
                , foundDancer.getName(),
                foundDancer.getCrewName(),
                foundDancer.getDanceLevel(),
                foundDancer.getGenres()));
    }
    w.write("</ul>");
    w.write("<a href=\"/chap02/dancer/register\">새로운 등록하기</a>");
    w.write("</body>\n");
    w.write("</html>");
    w.close();
}
```