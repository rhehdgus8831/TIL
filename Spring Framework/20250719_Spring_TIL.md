
# Spring Core IOC DI

## 1. IOC DI 예제

### Chapter 1: 수동 DI (Manual Dependency Injection)
```
com.spring.core.chap01/
├── manage/
│   └── HotelManager.java          # 수동 DI 구성자 (Assembler)
├── AsianRestaurant.java           # Restaurant 구현체
├── Chef.java                      # 셰프 인터페이스
├── Course.java                    # 코스 메뉴 인터페이스
├── FrenchCourse.java              # Course 구현체
├── Hotel.java                     # 최상위 도메인 객체
├── ItalianCourse.java             # Course 구현체
├── JannChef.java                  # Chef 구현체
├── KimuraChef.java                # Chef 구현체
├── Restaurant.java                # 레스토랑 인터페이스
├── StephaneChef.java              # Chef 구현체
├── SushiCourse.java               # Course 구현체
└── WesternRestaurant.java         # Restaurant 구현체
```

### Chapter 2-2: Spring IoC Container를 통한 자동 DI
```
com.spring.core.chap02_2/
├── config/
│   └── CarConfig.java             # Car 빈 설정
├── practice1/
│   ├── config/
│   │   └── LibraryConfig.java     # Library 빈 설정
│   ├── Book.java                  # 도서 모델
│   └── Library.java               # 라이브러리 서비스
├── practice2/
│   ├── config/
│   │   └── MusicConfig.java       # Music 빈 설정
│   ├── MusicPlayer.java           # 음악 플레이어
│   └── Speaker.java               # 스피커
└── vehicle/
    ├── Car.java                   # 자동차 서비스
    └── Engine.java                # 엔진 컴포넌트
```

## 2. IoC(Inversion of Control) 개념 완전 정리

### 2.1 IoC란 무엇인가?
**IoC(제어의 역전)**는 객체의 생성과 관리 책임을 개발자가 아닌 외부 컨테이너(Spring Container)가 담당하는 설계 원칙입니다.

전통적인 방식에서는 개발자가 직접 `new` 키워드를 사용해서 객체를 생성하고 의존성을 주입했다면, IoC 방식에서는 Spring Container가 이 모든 과정을 대신 해줍니다.

### 2.2 전통적인 방식 vs IoC 방식 비교

| 구분 | 전통적인 방식 | IoC 방식 |
|------|-------------|----------|
| 객체 생성 | 개발자가 직접 `new` 키워드로 생성 | Spring Container가 자동 생성 |
| 의존성 주입 | 개발자가 수동으로 주입 | Container가 자동 주입 |
| 생명주기 관리 | 개발자가 직접 관리 | Container가 통합 관리 |
| 제어 주체 | 개발자 | Spring Container |

## 3. DI(Dependency Injection) 개념 완전 정리

### 3.1 DI란 무엇인가?
**DI(의존성 주입)**는 객체가 필요로 하는 의존성을 외부에서 주입받는 디자인 패턴입니다.
이를 통해 객체 간의 결합도를 낮추고, 테스트 용이성과 코드 유연성을 높일 수 있습니다.

### 3.2 DI의 3가지 방법

| 방법 | 설명 | 장점 | 단점 |
|------|------|------|------|
| **생성자 주입** | 생성자를 통해 의존성 주입 | 불변성 보장, 필수 의존성 명시 | 생성자 코드 증가 |
| **세터 주입** | Setter 메서드를 통해 의존성 주입 | 선택적 의존성 가능 | 불변성 보장 어려움 |
| **필드 주입** | 필드에 직접 의존성 주입 | 코드 간결 | 테스트 어려움, 불변성 보장 어려움 |

## 4. Chapter 1 코드 분석 (수동 DI) - 원본 코드 포함

### 4.1 인터페이스 정의

#### Chef.java
```java
package com.spring.core.chap01;

public interface Chef {
    void cook();
}
```

#### Course.java
```java
package com.spring.core.chap01;

public interface Course {
    void combineMenu();
}
```

#### Restaurant.java
```java
package com.spring.core.chap01;

public interface Restaurant {
    void order();
}
```

### 4.2 Chef 인터페이스 구현체들

#### KimuraChef.java
```java
package com.spring.core.chap01;

public class KimuraChef implements Chef {
    @Override
    public void cook() {
        System.out.println("스시 니기리의 장인 키무라다.");
    }
}
```

#### JannChef.java
```java
package com.spring.core.chap01;

// 쉐프 클래스
public class JannChef implements Chef{

    // 요리 기능
    public void cook() {
        System.out.println("프랑스 요리의 대가 쟝발쟝입니다.");
    }
}
```

#### StephaneChef.java
```java
package com.spring.core.chap01;

public class StephaneChef implements Chef{

    public void cook() {
        System.out.println("이탈리안 요리의 대가 스테판입니다.");
    }
}
```

### 4.3 Course 인터페이스 구현체들

#### SushiCourse.java
```java
package com.spring.core.chap01;

public class SushiCourse implements Course{

    public void combineMenu() {
        System.out.println("====== 스시 코스 구성 ======");
        System.out.println("1. 대합 맑은국");
        System.out.println("2. 전어, 고등어, 도미 스시");
        System.out.println("3. 구이 요리");
        System.out.println("4. 오도로, 우니군함, 복어");
        System.out.println("5. 튀김 요리");
        System.out.println("6. 아나고덮밥");
        System.out.println("7. 조리장 특선 디저트");
    }
}
```

#### FrenchCourse.java
```java
package com.spring.core.chap01;

public class FrenchCourse implements Course{

    public void combineMenu() {
        System.out.println("====== 프렌치 코스 구성 ======");
        System.out.println("1. 제철 채소, 퀴노아");
        System.out.println("2. 트러플 크림 스프");
        System.out.println("3. 참돔 포와레");
        System.out.println("4. 한우 살치살 스테이크");
        System.out.println("5. 프렌치 랙");
        System.out.println("6. 조리장 특제 마카롱");
    }
}
```

#### ItalianCourse.java
```java
package com.spring.core.chap01;

public class ItalianCourse implements Course {
    @Override
    public void combineMenu() {
        System.out.println("====  이탈리안 코스 구성 ======");
    }
}
```

### 4.4 Restaurant 인터페이스 구현체들

#### AsianRestaurant.java
```java
package com.spring.core.chap01;

public class AsianRestaurant implements Restaurant{

    // 전문 셰프
    private Chef mainChef;
    // 코스 메뉴
    private Course course;

    // 생성자
    public AsianRestaurant(Chef chef, Course course) {
        this.mainChef = chef;
        this.course = course;
    }

    @Override
    public void order() {
        System.out.println("아시안 요리를 주문합니다.");
        course.combineMenu();
        mainChef.cook();
    }
}
```

#### WesternRestaurant.java
```java
package com.spring.core.chap01;

// 서양식 레스토랑
public class WesternRestaurant implements Restaurant{

    // 메인 쉐프 고용
    private Chef mainChef;

    // 요리 코스 구성
    private Course course;

    // 생성자
    public WesternRestaurant(Chef chef,Course course) {
        this.mainChef = chef;
        this.course = course;
    }

    // 주문 기능
    public void order() {
        System.out.println("서양 요리를 주문합니다.");
        course.combineMenu();
        mainChef.cook();
    }
}
```

### 4.5 최상위 도메인 객체

#### Hotel.java
```java
package com.spring.core.chap01;

// 호텔 설계도
public class Hotel {

    // 레스토랑 입점
    private final Restaurant restaurant;

    // 헤드셰프 고용
    private final Chef headChef;

//     생성자   IOC 개념 - 의존성 주입
    public Hotel(Restaurant restaurant,Chef chef) {
        this.restaurant = restaurant;
        this.headChef = chef;
    }

//    public void setRestaurant(Restaurant restaurant) {}
//    public void setHeadChef(Chef chef) {}

    // 레스토랑 예약 기능
    public  void reserve(){
        System.out.println("레스토랑을 예약합니다.");
        System.out.println("헤드 셰프명:" + headChef.getClass().getSimpleName());
        restaurant.order();
    }
}
```

### 4.6 수동 DI 구성자 (Assembler Pattern)

#### HotelManager.java
```java
package com.spring.core.chap01.manage;

import com.spring.core.chap01.*;

// 호텔의 레스토랑 입점과 헤드쉐프 고용은 내가 다 책임지고 하겠다.
public class HotelManager {

    // 셰프를 고용하는 기능 - 셰프 객체 생성을 위임
    public Chef chef() {
        return new KimuraChef();
    }

    // 코스를 개발하는 기능
    public Course course() {
        return new SushiCourse();
    }

    // 레스토랑을 입점하는 기능
    public Restaurant restaurant() {
        return new AsianRestaurant(chef(), course());
    }

    // 호텔의 의존객체를 조립해주는 기능
    public Hotel hotel() {
        return new Hotel(restaurant(), chef());
    }
}
```

### 4.7 테스트 코드

#### HotelTest.java
```java
package com.spring.chap01;

import com.spring.core.chap01.Hotel;
import com.spring.core.chap01.manage.HotelManager;
import org.junit.jupiter.api.Test;

class HotelTest {

    // 테스트 메서드
    @Test
    void hotel() {

        // 호텔객체를 생성 - 매니저한테 문의
        HotelManager manager = new HotelManager();

        Hotel hotel = manager.hotel();
        hotel.reserve();
    }
}
```

### 4.8 Chapter 1 코드 흐름 정리

#### 수동 DI 실행 흐름
1. **테스트 시작**: `HotelTest.hotel()` 메서드 실행
2. **HotelManager 생성**: `new HotelManager()` 객체 생성
3. **Hotel 객체 요청**: `manager.hotel()` 호출
4. **의존성 체인 생성**:
    - `HotelManager.chef()` → `KimuraChef` 객체 생성
    - `HotelManager.course()` → `SushiCourse` 객체 생성
    - `HotelManager.restaurant()` → `AsianRestaurant` 객체 생성 (chef, course 주입)
    - `HotelManager.hotel()` → `Hotel` 객체 생성 (restaurant, chef 주입)
5. **기능 실행**: `hotel.reserve()` 호출
6. **실행 흐름**:
    - `Hotel.reserve()` 실행
    - "레스토랑을 예약합니다." 출력
    - "헤드 셰프명:KimuraChef" 출력
    - `restaurant.order()` 호출 (AsianRestaurant)
    - "아시안 요리를 주문합니다." 출력
    - `course.combineMenu()` 호출 (SushiCourse)
    - 스시 코스 구성 내용 출력
    - `mainChef.cook()` 호출 (KimuraChef)
    - "스시 니기리의 장인 키무라다." 출력

#### 수동 DI의 특징
- **명시적 의존성 관리**: 개발자가 직접 모든 의존성을 생성하고 주입
- **중앙 집중식 관리**: HotelManager가 모든 객체 생성을 담당
- **컴파일 타임 안전성**: 의존성 문제를 컴파일 시점에 발견 가능
- **직관적 이해**: 객체 생성 과정이 명확하게 보임

## 5. Chapter 2-2 코드 분석 (Spring IoC Container) - 원본 코드 포함

### 5.1 Vehicle 예제 (기본 Spring DI)

#### Engine.java
```java
package com.spring.core.chap02_2.vehicle;

import org.springframework.stereotype.Component;

// @Component
public class Engine {

    public Engine() {
        System.out.println("엔진이 생성됨 !");
    }

    public void start() {
        System.out.println("엔진 시동이 걸렸습니다.");
    }
}
```

#### Car.java
```java
package com.spring.core.chap02_2.vehicle;

import org.springframework.stereotype.Component;

// @Component // 스프링이 관리
public class Car {

    // 의존 객체 설정
    private final Engine engine;

    // 생성자 주입 - 생성자를 통해 의존객체 엔진을 받아서 결합
    public Car(Engine engine) {
        this.engine = engine;
        System.out.println("자동차가 생성됨!");
    }

//    setter 주입 - setter를 통해 의존 객체를 전달 받아 결합
//    public void setEngine(Engine engine) {
//        this.engine = engine;
//    }

    public void drive() {
        engine.start();
        System.out.println("자동차가 달립니다.");
    }
}
```

#### CarConfig.java
```java
package com.spring.core.chap02_2.config;

// 스프링에게 관리할 객체를 알려주는 클래스

import com.spring.core.chap02_2.vehicle.Car;
import com.spring.core.chap02_2.vehicle.Engine;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration // 이거는 자바 클래스가 아니라 사실 설정파일임
public class CarConfig {

    // 스프링아 나 대신 엔진 좀 만들어라
    @Bean
    public Engine engine() {
        return new Engine();
    }

    // 스프링아 나 대신 차 좀 만들어 대신 그때 엔진을 결합해
    @Bean
    public Car car() {
        return new Car(engine()); // 생성자 주입
    }
}
```

#### CarTest.java
```java
package com.spring.core.chap02_2.vehicle;

import com.spring.core.SpringCor202507Application;
import com.spring.core.chap02_2.config.CarConfig;
import org.junit.jupiter.api.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

import static org.junit.jupiter.api.Assertions.*;

class CarTest {

    @Test
    void test1() {
        // Car car = new Car(new Engine());

        // 스프링은 등록된 모든 객체를 Bean이라는 개념으로 관리하며
        // 객체의 생성과 소멸까지의 모든 생애주기를 스프링이 통합 관리 운영함.
        // 운영하는 커더란 창고 ApplicationContext라고 부름
        ApplicationContext context
                = new AnnotationConfigApplicationContext(CarConfig.class);

        Car car = context.getBean(Car.class);
        car.drive();
    }
}
```

### 5.1.1 Vehicle 예제 코드 흐름 정리

#### Spring IoC 실행 흐름 (Car 예제)
1. **테스트 시작**: `CarTest.test1()` 메서드 실행
2. **ApplicationContext 생성**: `new AnnotationConfigApplicationContext(CarConfig.class)`
3. **Spring Container 초기화**:
    - `CarConfig` 클래스 스캔
    - `@Configuration` 어노테이션 인식
    - `@Bean` 메서드들 발견
4. **Bean 생성 순서**:
    - `engine()` 메서드 실행 → `Engine` 객체 생성 → "엔진이 생성됨 !" 출력
    - `car()` 메서드 실행 → `Car` 객체 생성 (engine 주입) → "자동차가 생성됨!" 출력
5. **Bean 등록**: Spring Container에 Engine, Car Bean 등록
6. **Bean 조회**: `context.getBean(Car.class)` 호출
7. **기능 실행**: `car.drive()` 호출
8. **실행 흐름**:
    - `engine.start()` 호출 → "엔진 시동이 걸렸습니다." 출력
    - "자동차가 달립니다." 출력

#### Spring IoC의 특징 (Car 예제)
- **자동 의존성 관리**: Spring Container가 Engine과 Car의 의존성을 자동 해결
- **생성자 주입**: Car 생성자에 Engine을 자동 주입
- **싱글톤 관리**: 같은 타입의 Bean은 하나만 생성되어 재사용
- **생명주기 관리**: 객체의 생성과 소멸을 Spring이 관리

### 5.2 Library 예제 (Practice 1)

#### Book.java
```java
package com.spring.core.chap02_2.pracitce1;

import lombok.*;

import java.util.List;

@Getter
@Setter
@ToString
@EqualsAndHashCode
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Book {

    private String title;
    private String author;
}
```

#### Library.java
```java
package com.spring.core.chap02_2.pracitce1;

import java.util.List;
import java.util.Stack;

public class Library {

    private List books;

    public Library(List books) {
        this.books = books;
    }

    public void printBooks() {
        books.forEach(System.out::println);
    }
}
```

#### LibraryConfig.java
```java
package com.spring.core.chap02_2.pracitce1.config;

import com.spring.core.chap02_2.pracitce1.Book;
import com.spring.core.chap02_2.pracitce1.Library;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

@Configuration
public class LibraryConfig {

    // 책 목록을 나대신 스프링이 만들어서 관리해줘
    @Bean
    public List books() {
        return List.of(
                new Book("캐치캐치 티니핑", "하츄핑"),
                new Book("뽀롱뽀롱 뽀로로", "삘리리"),
                new Book("반지의 제왕", "링링링")
        );
    }

    // 도서관도 나대신 만들어주고 책을 생성자로 주입해줘
    @Bean
    public Library library() {
        return new Library(books());
    }
}
```

#### LibraryTest.java
```java
package com.spring.core.chap02_2.pracitce1;

import com.spring.core.chap02_2.pracitce1.config.LibraryConfig;
import com.spring.core.chap02_2.practice2.config.MusicConfig;
import org.junit.jupiter.api.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

import static org.junit.jupiter.api.Assertions.*;

class LibraryTest {

    @Test
    void test1() {

        ApplicationContext context
                = new AnnotationConfigApplicationContext(LibraryConfig.class);

        Library library = context.getBean(Library.class);
        library.printBooks();
    }
}
```

### 5.2.1 Library 예제 코드 흐름 정리

#### Spring IoC 실행 흐름 (Library 예제)
1. **테스트 시작**: `LibraryTest.test1()` 메서드 실행
2. **ApplicationContext 생성**: `new AnnotationConfigApplicationContext(LibraryConfig.class)`
3. **Spring Container 초기화**:
    - `LibraryConfig` 클래스 스캔
    - `@Configuration` 어노테이션 인식
    - `@Bean` 메서드들 발견
4. **Bean 생성 순서**:
    - `books()` 메서드 실행 → `List` 객체 생성 (3개의 Book 객체 포함)
    - `library()` 메서드 실행 → `Library` 객체 생성 (books 주입)
5. **Bean 등록**: Spring Container에 List, Library Bean 등록
6. **Bean 조회**: `context.getBean(Library.class)` 호출
7. **기능 실행**: `library.printBooks()` 호출
8. **실행 흐름**:
    - `books.forEach(System.out::println)` 실행
    - 각 Book 객체의 toString() 메서드 호출 (Lombok @ToString)
    - 3개의 책 정보 출력

#### Library 예제의 특징
- **컬렉션 Bean 관리**: Spring이 List을 하나의 Bean으로 관리
- **Lombok 활용**: @ToString, @AllArgsConstructor 등으로 코드 간소화
- **생성자 주입**: Library 생성자에 List을 자동 주입
- **메서드 체이닝**: List.of()와 forEach() 등 모던 Java 기능 활용

### 5.3 MusicPlayer 예제 (Practice 2)

#### Speaker.java
```java
package com.spring.core.chap02_2.practice2;

public class Speaker {

    public void playSound() {
        System.out.println("스피커에서 음악이 재생됩니다.");
    }
}
```

#### MusicPlayer.java
```java
package com.spring.core.chap02_2.practice2;

public class MusicPlayer {

    private Speaker speaker;

    public void setSpeaker(Speaker speaker) {
        this.speaker = speaker;
    }

    public void playMusic() {
        if(speaker == null){
            System.out.println("스피커가 셋팅되지 않았습니다.");
            return;
        }
        speaker.playSound();
        System.out.println("음악 플레이어에서 음악이 재생됩니다.");
    }
}
```

#### MusicConfig.java
```java
package com.spring.core.chap02_2.practice2.config;

import com.spring.core.chap02_2.practice2.MusicPlayer;
import com.spring.core.chap02_2.practice2.Speaker;
import org.springframework.context.annotation.Bean;

public class MusicConfig {

  @Bean
    public Speaker speaker() {
      return new Speaker();
  }
  @Bean
    public MusicPlayer musicPlayer() {
      MusicPlayer musicPlayer = new MusicPlayer();
      musicPlayer.setSpeaker(speaker());
      return musicPlayer;
  }
}
```

#### MusicPlayerTest.java
```java
package com.spring.core.chap02_2.practice2;

import com.spring.core.chap02_2.practice2.config.MusicConfig;
import org.junit.jupiter.api.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

import static org.junit.jupiter.api.Assertions.*;

class MusicPlayerTest {

    @Test
    void test() {
        ApplicationContext context
                = new AnnotationConfigApplicationContext(MusicConfig.class);

        MusicPlayer musicPlayer = context.getBean(MusicPlayer.class);
        musicPlayer.playMusic();
    }
}
```

### 5.3.1 MusicPlayer 예제 코드 흐름 정리

#### Spring IoC 실행 흐름 (MusicPlayer 예제)
1. **테스트 시작**: `MusicPlayerTest.test()` 메서드 실행
2. **ApplicationContext 생성**: `new AnnotationConfigApplicationContext(MusicConfig.class)`
3. **Spring Container 초기화**:
    - `MusicConfig` 클래스 스캔
    - `@Bean` 메서드들 발견 (주의: @Configuration 어노테이션 누락)
4. **Bean 생성 순서**:
    - `speaker()` 메서드 실행 → `Speaker` 객체 생성
    - `musicPlayer()` 메서드 실행:
        - `MusicPlayer` 객체 생성
        - `setSpeaker(speaker())` 호출 → Setter 주입
5. **Bean 등록**: Spring Container에 Speaker, MusicPlayer Bean 등록
6. **Bean 조회**: `context.getBean(MusicPlayer.class)` 호출
7. **기능 실행**: `musicPlayer.playMusic()` 호출
8. **실행 흐름**:
    - `speaker == null` 체크 → false (정상 주입됨)
    - `speaker.playSound()` 호출 → "스피커에서 음악이 재생됩니다." 출력
    - "음악 플레이어에서 음악이 재생됩니다." 출력

#### MusicPlayer 예제의 특징
- **Setter 주입**: 생성자 주입 대신 Setter 메서드를 통한 의존성 주입
- **null 체크**: 의존성 주입 실패 시 안전한 예외 처리
- **@Configuration 누락**: 설정 클래스에 @Configuration 어노테이션이 없음 (실제로는 필요)
- **수동 의존성 설정**: Bean 메서드 내에서 수동으로 의존성 주입

## 6. 핵심 Spring 애노테이션 완전 정리

### 6.1 구성 관련 애노테이션

| 애노테이션 | 역할 | 사용 위치 | 주요 속성 |
|-----------|------|----------|----------|
| `@Configuration` | Spring 설정 클래스임을 명시 | 클래스 | - |
| `@Bean` | Spring Container에 객체를 등록 | 메서드 | `name` (Bean 이름) |
| `@Component` | Spring Bean으로 등록 | 클래스 | `value` (Bean 이름) |

### 6.2 ApplicationContext 주요 메서드

| 메서드 | 역할 | 반환 타입 | 사용 예시 |
|--------|------|----------|----------|
| `getBean(Class)` | 타입으로 Bean 조회 | T | `context.getBean(Car.class)` |
| `getBean(String)` | 이름으로 Bean 조회 | Object | `context.getBean("car")` |
| `getBean(String, Class)` | 이름과 타입으로 Bean 조회 | T | `context.getBean("car", Car.class)` |

## 7. 수동 DI vs Spring IoC 완전 비교

### 7.1 코드 복잡성 비교

**수동 DI (Chapter 1)**
```java
// HotelManager.java - 개발자가 모든 의존성을 수동으로 관리
public class HotelManager {
    public Chef chef() {
        return new KimuraChef();                      // 직접 생성
    }
    
    public Course course() {
        return new SushiCourse();                     // 직접 생성
    }
    
    public Restaurant restaurant() {
        return new AsianRestaurant(chef(), course()); // 직접 주입
    }
    
    public Hotel hotel() {
        return new Hotel(restaurant(), chef());       // 직접 주입
    }
}
```

**Spring IoC (Chapter 2-2)**
```java
// CarConfig.java - Spring Container가 자동으로 관리
@Configuration
public class CarConfig {
    @Bean
    public Engine engine() {
        return new Engine();
    }
    
    @Bean
    public Car car() {
        return new Car(engine()); // Spring이 자동으로 Bean 생성 및 주입
    }
}
```

### 7.2 DI 방법 비교

**생성자 주입 (Car 예제)**
```java
// 생성자 주입 - 생성자를 통해 의존객체 엔진을 받아서 결합
public Car(Engine engine) {
    this.engine = engine;
    System.out.println("자동차가 생성됨!");
}
```

**Setter 주입 (MusicPlayer 예제)**
```java
// Setter 주입 - setter를 통해 의존 객체를 전달 받아 결합
public void setSpeaker(Speaker speaker) {
    this.speaker = speaker;
}
```

### 7.3 전체 흐름 비교

#### 수동 DI 전체 흐름
1. **개발자가 직접 제어**: 모든 객체 생성과 의존성 주입을 개발자가 명시적으로 처리
2. **순차적 생성**: HotelManager의 메서드를 통해 순차적으로 객체 생성
3. **명시적 의존성**: 어떤 객체가 어떤 의존성을 가지는지 코드에서 명확히 보임
4. **컴파일 타임 안전성**: 의존성 문제를 컴파일 시점에서 발견 가능

#### Spring IoC 전체 흐름
1. **Container가 제어**: Spring Container가 객체 생성과 의존성 주입을 자동 처리
2. **설정 기반 생성**: @Configuration과 @Bean을 통한 설정 기반 객체 생성
3. **자동 의존성 해결**: Spring이 의존성 그래프를 분석해서 자동으로 주입
4. **런타임 의존성 해결**: 실행 시점에 의존성 문제 발견 가능

## 8. 실행 결과 예상

### 8.1 Chapter 1 실행 결과 (HotelTest)
```
레스토랑을 예약합니다.
헤드 셰프명:KimuraChef
아시안 요리를 주문합니다.
====== 스시 코스 구성 ======
1. 대합 맑은국
2. 전어, 고등어, 도미 스시
3. 구이 요리
4. 오도로, 우니군함, 복어
5. 튀김 요리
6. 아나고덮밥
7. 조리장 특선 디저트
스시 니기리의 장인 키무라다.
```

### 8.2 Chapter 2-2 실행 결과

**CarTest 실행 결과**
```
엔진이 생성됨 !
자동차가 생성됨!
엔진 시동이 걸렸습니다.
자동차가 달립니다.
```

**LibraryTest 실행 결과**
```
Book(title=캐치캐치 티니핑, author=하츄핑)
Book(title=뽀롱뽀롱 뽀로로, author=삘리리)
Book(title=반지의 제왕, author=링링링)
```

**MusicPlayerTest 실행 결과**
```
스피커에서 음악이 재생됩니다.
음악 플레이어에서 음악이 재생됩니다.
```

## 9. 핵심 학습 포인트

### 9.1 IoC의 핵심 개념
- **제어의 역전**: 객체 생성 및 관리 책임이 개발자에서 Spring Container로 역전됨
- **ApplicationContext**: Spring의 핵심 IoC 컨테이너, 모든 Bean을 생성하고 관리
- **Bean 생명주기**: Spring이 객체의 생성, 초기화, 소멸까지 통합 관리
- **싱글톤 패턴**: 기본적으로 모든 Bean은 싱글톤으로 관리됨

### 9.2 DI의 핵심 개념
- **의존성 주입**: 필요한 의존성을 외부에서 주입받아 객체 간 결합도 감소
- **생성자 주입**: 가장 권장되는 DI 방식, 불변성과 필수 의존성을 보장
- **Setter 주입**: 선택적 의존성 주입에 유용, 런타임에 의존성 변경 가능
- **@Bean**: 메서드 레벨에서 Spring Container에 객체 등록

### 9.3 Spring의 핵심 장점
1. **결합도 감소**: 인터페이스 기반 설계로 구현체 변경이 용이
2. **테스트 용이성**: Mock 객체 주입으로 단위 테스트 간소화
3. **코드 재사용성**: 컴포넌트 기반 설계로 다양한 환경에서 재사용 가능
4. **유지보수성**: 설정을 통한 의존성 관리로 코드 변경 최소화
5. **생산성 향상**: 반복적인 객체 생성 코드 제거로 비즈니스 로직에 집중

### 9.4 실무 적용 팁
1. **@Configuration과 @Bean**: 복잡한 객체 생성 로직이 필요할 때 사용
2. **생성자 주입 우선**: Setter 주입보다 생성자 주입을 우선적으로 사용
3. **null 체크**: MusicPlayer 예제처럼 의존성 null 체크로 안전성 확보
4. **Lombok 활용**: Book 클래스처럼 @Getter, @Setter 등으로 코드 간소화

### 9.5 코드 흐름 이해의 중요성
- **수동 DI**: 개발자가 직접 제어하는 명시적 흐름으로 학습 초기에 이해하기 쉬움
- **Spring IoC**: Container가 자동으로 처리하는 흐름으로 설정 방식 이해 필요
- **단계별 학습**: 수동 DI로 개념을 익힌 후 Spring IoC로 발전하는 것이 효과적
- **디버깅 능력**: 각 단계의 흐름을 이해해야 문제 발생 시 원인 파악 가능
