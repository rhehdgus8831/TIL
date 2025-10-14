## Docker : Docker Compose와 Spring Boot Dockerizing

---

### ✅ 1. 스프링 부트 애플리케이션 도커라이즈 (Dockerizing)

*   **도커라이징(Dockerizing)**이란 우리의 애플리케이션(e.g., Spring Boot)을 도커 컨테이너에서 실행할 수 있도록 **도커 이미지로 만드는 전체 과정**을 의미합니다.

#### ➕ 도커라이징 절차

1.  **애플리케이션 빌드**: 먼저, Spring Boot 프로젝트를 빌드하여 실행 가능한 **JAR 파일**을 생성합니다.
    ```bash
    ./gradlew build
    # build/libs/your-app-name-0.0.1-SNAPSHOT.jar 파일 생성 확인
    ```

2.  **`Dockerfile` 작성**: 프로젝트의 루트 디렉토리에 도커 이미지를 만들기 위한 **설계도(`Dockerfile`)**를 작성합니다.
    ```dockerfile
    # 1. 베이스 이미지 선택: Java 17 환경이 설치된 경량 리눅스 이미지
    FROM openjdk:17-jdk-slim

    # 2. JAR 파일이 복사될 컨테이너 내 경로를 인자로 받음
    ARG JAR_FILE_PATH=build/libs/*.jar

    # 3. 호스트의 JAR 파일을 컨테이너의 app.jar 라는 이름으로 복사
    COPY ${JAR_FILE_PATH} app.jar

    # 4. 컨테이너가 시작될 때 실행할 명령어 설정
    ENTRYPOINT ["java", "-jar", "/app.jar"]
    ```
    *   **`ENTRYPOINT` vs `CMD`**: `CMD`는 컨테이너 실행 시 다른 명령어로 쉽게 덮어쓸 수 있는 기본값인 반면, `ENTRYPOINT`는 컨테이너의 주 실행 명령어로 고정되어 더 안정적입니다.

3.  **도커 이미지 빌드**: `Dockerfile`이 있는 위치에서 `docker build` 명령어를 실행하여 이미지를 생성합니다.
    ```bash
    # -t 옵션: 이미지에 이름과 태그(버전)를 부여 (이름:태그)
    # . (점): 현재 디렉토리의 Dockerfile을 사용하라는 의미
    docker build -t my-spring-app:0.0.1 .
    ```

4.  **도커 컨테이너 실행**: 생성된 이미지를 `docker run` 명령어로 실행하여 컨테이너를 띄웁니다.
    ```bash
    # -d: 백그라운드에서 실행
    # -p 8080:8080: 호스트의 8080 포트와 컨테이너의 8080 포트를 연결(매핑)
    # --name: 컨테이너에 이름 부여
    docker run -d -p 8080:8080 --name my-spring-container my-spring-app:0.0.1
    ```

---

### ✅ 2. 도커 컴포즈 (Docker Compose): 멀티 컨테이너 관리의 시작

*   **문제점**: 현대적인 애플리케이션은 대부분 웹 서버, 데이터베이스, 캐시 서버 등 여러 개의 컨테이너가 함께 동작해야 합니다. 이 컨테이너들을 `docker run` 명령어로 하나씩 수동으로 실행하고, 서로 네트워크로 연결하며, 환경 변수를 주입하는 것은 매우 번거롭고 실수가 발생하기 쉽습니다.

*   **도커 컴포즈(Docker Compose)**는 **여러 개의 컨테이너**로 구성된 애플리케이션을 **하나의 YAML 파일**로 정의하고, **단 하나의 명령어**로 실행, 중지, 관리할 수 있게 해주는 도구입니다.

#### ➕ `docker-compose.yml` 파일의 핵심 구조

*   `docker-compose.yml` 파일은 멀티 컨테이너 애플리케이션의 **전체 설계도** 역할을 합니다.

| 키워드 | 설명 |
| :--- | :--- |
| **`version`** | 사용할 도커 컴포즈 파일의 버전을 명시합니다. (e.g., `'3.8'`) |
| **`services`** | 실행할 **컨테이너들의 목록**을 정의하는 최상위 키입니다. 각 서비스가 하나의 컨테이너에 해당합니다. |
| **`image`** | 컨테이너를 생성할 때 사용할 도커 이미지를 지정합니다. (e.g., `mariadb:10.5`) |
| **`build`** | 이미지를 직접 빌드할 경우, `Dockerfile`이 있는 경로를 지정합니다. (`image`와 `build` 중 하나만 사용) |
| **`container_name`** | 컨테이너의 이름을 지정합니다. |
| **`ports`** | 호스트와 컨테이너 간의 포트를 매핑합니다. (`HOST:CONTAINER`) |
| **`environment`** | 컨테이너 내부에 설정할 **환경 변수**를 정의합니다. (e.g., DB 비밀번호) |
| **`volumes`** | 호스트의 디렉토리와 컨테이너의 디렉토리를 연결(마운트)하여 데이터를 **영속적으로 저장**합니다. 컨테이너가 삭제되어도 데이터가 유지됩니다. |
| **`depends_on`** | 서비스 간의 **실행 순서**를 정의합니다. (e.g., `spring-app`은 `db`가 먼저 실행된 후에 시작) |
| **`networks`** | 컨테이너들이 속할 가상 네트워크를 정의합니다. 같은 네트워크에 속한 컨테이너들은 **서비스 이름**으로 서로를 호출할 수 있습니다. |

#### ➕ Spring Boot + MariaDB `docker-compose.yml` 예시

```yaml
version: '3.8'

services:
  # MariaDB 데이터베이스 서비스 정의
  db:
    image: mariadb:10.5
    container_name: my-mariadb
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: mydatabase
      MYSQL_USER: myuser
      MYSQL_PASSWORD: mypassword
    volumes:
      - ./db-data:/var/lib/mysql # 호스트의 ./db-data와 컨테이너의 mysql 데이터 디렉토리 연결
    networks:
      - my-network

  # Spring Boot 애플리케이션 서비스 정의
  spring-app:
    build: . # 현재 디렉토리의 Dockerfile로 이미지 빌드
    container_name: my-spring-app
    ports:
      - "8080:8080"
    environment:
      # 중요: 'db'라는 서비스 이름으로 DB에 접근
      DB_HOST: db 
      DB_USERNAME: myuser
      DB_PASSWORD: mypassword
    depends_on:
      - db # db 서비스가 먼저 시작되어야 함
    networks:
      - my-network

networks:
  my-network:
    driver: bridge
```

#### ➕ 도커 컴포즈 명령어

*   **`docker-compose up`**: `docker-compose.yml` 파일을 읽어 모든 서비스를 생성하고 시작합니다. (`-d` 옵션으로 백그라운드 실행)
*   **`docker-compose down`**: 모든 서비스를 중지하고 컨테이너, 네트워크를 제거합니다. (`-v` 옵션으로 볼륨까지 제거)
*   **`docker-compose ps`**: 현재 실행 중인 서비스들의 상태를 확인합니다.

---

### 📌 요약

*   **도커라이징**은 Spring Boot 애플리케이션을 **`Dockerfile`**을 통해 **도커 이미지**로 만드는 과정입니다.
*   **도커 컴포즈**는 `docker-compose.yml`이라는 **하나의 설정 파일**을 사용하여, **여러 개의 컨테이너**로 구성된 애플리케이션의 전체 스택(e.g., 웹 서버 + DB)을 **한 번에 관리**하는 강력한 도구입니다.
*   도커 컴포즈의 **네트워크** 기능을 통해, 컨테이너들은 IP 주소가 아닌 **서비스 이름**(e.g., `db`)으로 서로를 쉽게 참조할 수 있습니다.
*   **볼륨(volumes)**을 사용하여 데이터베이스 파일과 같은 중요한 데이터를 컨테이너 외부(호스트)에 **영속적으로 저장**해야 합니다.
