# **Spring 파일 업로드 및 YAML 설정**

오늘은 스프링 부트 애플리케이션에서 파일 업로드를 처리하기 위한 필수 설정들을 학습했다.
또한, 기존의 `.properties` 설정 파일을 계층 구조로 더 쉽게 읽을 수 있는 **YML(`.yml`)** 형식으로 변경하는 방법을 익혔다.

-----

## **1. YML 설정 파일 (`application.yml`)**

**YML(YML Ain't Markup Language)** 은 사람이 읽기 쉬운 데이터 직렬화 형식이다. `.properties`의 `key.key.key=value` 방식과 달리, **들여쓰기**를 통해 데이터의 계층 구조를 표현하므로 가독성이 높고 관리가 편하다.

### **`application.yml` 전체 코드 및 분석**

```yaml
server:
  port: 9000

spring:
  application:
    name: spring-web-basic202507
  mvc:
    view:
      prefix: /WEB-INF/views/
      suffix: .jsp
      
  # file upload setting
  # 파일 업로드 관련 설정
  servlet:
    multipart:
      max-file-size: 10MB  # 업로드할 파일 1개의 최대 용량을 10MB로 제한
      max-request-size: 100MB  # 한번에 업로드할 수 있는 파일들의 총 용량을 100MB로 제한

# custom setting
# 사용자 정의 설정: 파일 저장 루트 경로
file:
  upload:
    # ${user.home} 은 시스템의 사용자 홈 디렉토리 경로를 의미
    location: ${user.home}/spring/upload/

# 로그 레벨 설정
logging:
  level:
    root: INFO
    # com.spring.basic 패키지 하위의 로그는 DEBUG 레벨까지 모두 출력
    com.spring.basic: DEBUG
```

-----

## **2. 파일 업로드 자바 설정 (Java Configuration)**

`application.yml`에 정의된 설정값들을 자바 클래스에서 읽어와, 파일 업로드에 필요한 환경을 구성한다.

### **2-1. 업로드 경로 설정: `FileUploadConfig.java`**

* **개념**: `application.yml`에 정의한 커스텀 프로퍼티(`file.upload.location`) 값을 읽어와서, 애플리케이션이 시작될 때 해당 경로의 디렉터리가 실제로 존재하도록 보장하는 역할을 한다.

* **주요 어노테이션**:

    * **`@Configuration`**: 이 클래스가 스프링의 설정 파일임을 명시한다.
    * **`@Value("${프로퍼티.키}")`**: `.yml` 파일의 프로퍼티 키를 통해 값을 읽어와서 필드에 주입한다.
    * **`@PostConstruct`**: 빈(Bean)이 생성되고 의존성 주입이 완료된 후, **딱 한 번 자동으로 실행**되는 초기화 메서드를 지정한다.

<!-- end list -->

```java
package com.spring.basic.chap8.config;

import jakarta.annotation.PostConstruct;
import lombok.Getter;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import java.io.File;

// 파일 업로드 관련 설정 클래스 (필수 설정)
@Configuration
@Getter
public class FileUploadConfig {

    // application.yml에 있는 파일업로드 루트경로를 읽어옴
    @Value("${file.upload.location}") // yml에서 값을 읽어서 location 필드에 주입
    private String location;

    // 루트 경로가 있는지 확인하고 없으면 디렉토리를 생성
    @PostConstruct   // 이 빈이 생성되는 순간 자동으로 1회 실행
    public void init() {
        File directory = new File(location);
        // 지정된 경로에 디렉터리가 존재하지 않으면
        if (!directory.exists()) {
            // 디렉터리를 생성하라
            directory.mkdirs();
        }
    }
}
```

### **2-2. 외부 자원 접근 설정: `WebResourceConfig.java`**

* **개념**: 서버의 로컬 디스크에 저장된 업로드 파일들은 기본적으로 외부에서 URL로 직접 접근할 수 없다. 이 설정은 특정 URL 요청이 들어왔을 때, 서버의 특정 폴더에 있는 파일을 찾아 보여주도록 **URL과 실제 파일 경로를 연결**해주는 역할을 한다.

* **`WebMvcConfigurer`**: 스프링 MVC의 설정을 개발자가 직접 커스터마이징할 수 있도록 도와주는 인터페이스다.

<!-- end list -->

```java
package com.spring.basic.chap8.config;

import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

// 사용자가 업로드한 파일 서버에 URL로 접근할 수 있게 해주는 설정 (필수 설정)
@Configuration
@RequiredArgsConstructor
public class WebResourceConfig implements WebMvcConfigurer {

    // 파일 업로드 경로가 담긴 설정 객체를 주입받음
    private final FileUploadConfig fileUploadConfig;
    
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        /*
            파일서버에 저장된 C:/Users/user/spring/upload 안에 들어있는 파일들을
            백엔드 서버에서 http://localhost:9000/uploads/파일명 으로 요청했을 때 꺼내주겠다.
        */
        
        // 1. URL 패턴 지정: /uploads/ 로 시작하는 모든 요청을 처리
        registry.addResourceHandler("/uploads/**")
                // 2. 실제 파일이 저장된 로컬 경로 지정
                // "file:" 접두어는 로컬 파일 시스템의 경로임을 명시
                .addResourceLocations("file:" + fileUploadConfig.getLocation());
    }
}
```

* **동작 예시**: 클라이언트가 `http://localhost:9000/uploads/cat.jpg`를 요청하면, 스프링은 이 설정을 보고 서버의 `C:/Users/사용자명/spring/upload/cat.jpg` 파일을 찾아서 응답으로 보내준다.