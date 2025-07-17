# SOLID 원칙


## 1. S ‒ 단일 책임 원칙 (Single Responsibility Principle)

| 한 줄 정의 | "클래스(또는 모듈)는 **단 하나의 변경 이유**만 가져야 한다." |
|-----------|---------------------------------------------------------|
| 왜? | 서로 다른 책임이 섞이면 수정 범위가 기하급수로 커짐 |
| 체크 포인트 | - 파일 이름만 보고 역할이 떠오르는가? <br>- "XXService" 안에 여러 _Manager_, _Util_ 코드가 섞여 있지 않은가? |

````
// BAD: 여러 책임이 섞임
class Employee {
void calculateSalary() { /* 급여 계산 */ }
void generateReport() { /* 보고서 생성 */ }
}

// GOOD: 각자 하나의 책임만
class SalaryCalculator {
double calculate(Employee emp) { /* 급여 계산 */ }
}
class ReportGenerator {
String generate(Employee emp) { /* 보고서 생성 */ }
}
````

---

## 2. O ‒ 개방·폐쇄 원칙 (Open/Closed Principle)

| 한 줄 정의 | "기존 코드는 **닫고**, 새 기능은 **확장**으로 해결한다." |
|-----------|----------------------------------------------|
| 비유 | 스마트폰 본체(코드) ↔ 앱 추가(확장) |
| 실천 팁 | - 변하는 부분을 **전략 객체**로 빼서 주입 <br>- `switch` 나 `if-else`가 새 타입마다 늘어난다면 위험 신호 |


 ````
// BAD: 새 도형마다 코드 수정 필요
class AreaCalculator {
    double calculate(Rectangle r) { /* ... */ }
    double calculate(Circle c) { /* ... */ }
    // 삼각형 추가 시 또 수정해야 함!
}

// GOOD: 인터페이스로 확장 가능하게
interface Shape {
    double calculateArea();
}
class Rectangle implements Shape { /* ... */ }
class Circle implements Shape { /* ... */ }
class AreaCalculator {
    double calculate(Shape shape) {
        return shape.calculateArea(); // 수정 없이 확장!
    }
}
````

---

## 3. L ‒ 리스코프 치환 원칙 (Liskov Substitution Principle)

| 한 줄 정의 | "자식은 **부모 대신** 써도 **동일하게** 동작해야 한다." |
|-----------|------------------------------------------|
| 핵심 | 상속(또는 구현) 관계가 **행동 계약**을 깨뜨리면 안 됨 |
| 적신호 | 오버라이드 후 예외·의미 변경, 파라미터 제한 강화 등 |


````

// BAD: 정사각형이 직사각형 규약을 깨뜨림
Rectangle r = new Square();
r.setWidth(5);
r.setHeight(4);
// 기대: 20, 실제: 16 (LSP 위반!)

// GOOD: 부모 계약을 그대로 유지
List<String> list = new ArrayList<>(); // 또는 LinkedList
list.add("item"); // 어떤 구현체든 동일하게 동작

````
---

## 4. I ‒ 인터페이스 분리 원칙 (Interface Segregation Principle)

| 한 줄 정의 | "뚱뚱한 인터페이스 하나보다 **날씬한 여러 개**가 낫다." |
|-----------|----------------------------------------------|
| 왜? | 불필요한 메서드 구현 강요 → 의존성·컴파일 영향 증가 |
| 가이드 | 기능별로 쪼개고 **필요한 것만** 구현하게 하기 |

````
// BAD: 로봇이 먹는 기능을 강제로 구현
interface Worker {
    void work();
    void eat(); // 로봇은 안 먹는데...
}

// GOOD: 필요한 것만 구현
interface Workable { void work(); }
interface Eatable { void eat(); }
class Human implements Workable, Eatable { /* ... */ }
class Robot implements Workable { /* eat() 구현 안 해도 됨! */ }
````

---

## 5. D ‒ 의존성 역전 원칙 (Dependency Inversion Principle)

| 한 줄 정의 | "**구체** 대신 **추상**(Interface, 추상클래스)에 의존하라." |
|-----------|--------------------------------------------------|
| 효과 | 모듈 교체·테스트가 쉬워짐, OCP 달성에 핵심 |
| 실천 | - 모든 의존 관계의 타입을 **인터페이스**로 선언 <br>- 객체 생성은 **팩토리**나 **DI 컨테이너**에 위임 |
````
// BAD: 구체 클래스에 의존
class Computer {
    private Keyboard keyboard = new Keyboard(); // 교체 불가!
}

// GOOD: 추상에 의존
class Computer {
    private InputDevice inputDevice; // 인터페이스 타입
    public Computer(InputDevice device) { // 주입받음
        this.inputDevice = device;
    }
}
// 키보드든 마우스든 자유롭게 교체 가능!
````

---

## 한눈에 보는 SOLID

| 원칙 | 키워드 | 깨졌을 때 흔한 증상 |
|------|--------|--------------------|
| S | 하나의 책임 | "이 파일 건드리면 다른 기능도 터져요" |
| O | 확장·수정 분리 | 새 요구마다 원본 코드 대수술 |
| L | 대체 가능성 | 자식으로 바꿨더니 버그 폭발 |
| I | 인터페이스 쪼개기 | 빈 메서드 `// TODO` 남발 |
| D | 추상에 의존 | 단위 테스트하려고 전체 시스템 기동 |

---

"**단일·확장·치환·분리·역전**"  
→ 클래스는 하나만, 기능은 확장, 자식은 부모처럼, 인터페이스는 얇게, 의존은 추상에!
