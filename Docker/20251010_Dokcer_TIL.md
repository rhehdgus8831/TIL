## Docker, AWS, and CI/CD with GitHub Actions

---

### ✅ 1. Docker Hub: Docker 이미지의 중앙 저장소

*   **Docker Hub**는 Docker, Inc.에서 제공하는 클라우드 기반의 **Docker 이미지 레지스트리(Registry)**입니다. 간단히 말해, **"Docker 이미지를 위한 GitHub"**라고 생각할 수 있습니다.

*   **핵심 역할**:
    1.  **저장(Store)**: `Dockerfile`로 빌드한 나만의 커스텀 이미지를 안전하게 저장합니다.
    2.  **공유(Share)**: 팀원이나 다른 개발자들과 이미지를 쉽게 공유할 수 있습니다.
    3.  **배포(Distribute)**: CI/CD 파이프라인에서 빌드된 이미지를 Docker Hub에 푸시(Push)하면, 실제 운영 서버(EC2)에서는 이 이미지를 풀(Pull)하여 애플리케이션을 실행합니다.

*   **주요 명령어**:
    *   `docker login`: Docker Hub 계정으로 로그인합니다.
    *   `docker tag <로컬 이미지> <사용자명>/<이미지명>:<태그>`: 이미지를 Docker Hub에 푸시하기 위한 이름 형식으로 변경합니다.
    *   `docker push <사용자명>/<이미지명>:<태그>`: Docker Hub로 이미지를 업로드합니다.

---

### ✅ 2. AWS EC2에 Docker 환경 구축하기

*   AWS EC2 인스턴스는 기본적으로 비어있는 가상 서버이므로, Docker 컨테이너를 실행하기 위해서는 먼저 **Docker 실행 환경을 구축**해야 합니다.

*   **구축 절차**:
    1.  **EC2 인스턴스 생성**: Amazon Linux 2 또는 Ubuntu와 같은 OS를 선택하여 인스턴스를 생성합니다.
    2.  **SSH 접속**: 키 페어를 사용하여 인스턴스에 원격으로 접속합니다.
    3.  **Docker 설치**: 각 OS에 맞는 명령어로 Docker 엔진을 설치합니다.
        ```bash
        # Amazon Linux 2 예시
        sudo yum update -y
        sudo yum install -y docker
        ```
    4.  **Docker 서비스 시작**: Docker 데몬을 실행하고, 시스템 부팅 시 자동으로 시작되도록 설정합니다.
        ```bash
        sudo systemctl start docker
        sudo systemctl enable docker
        ```
    5.  **사용자 권한 설정**: `sudo` 없이 `docker` 명령어를 실행할 수 있도록 현재 사용자(e.g., `ec2-user`)를 `docker` 그룹에 추가합니다. (보안상 주의가 필요하지만 개발/학습 환경에서는 편의성을 위해 사용)
        ```bash
        sudo usermod -aG docker ec2-user
        # 권한 적용을 위해 SSH 세션을 재접속해야 합니다.
        ```

---

### ✅ 3. CI/CD 파이프라인의 이해

*   **CI/CD**는 **"지속적인 통합(Continuous Integration)"**과 **"지속적인 배포(Continuous Deployment/Delivery)"**를 의미하며, 애플리케이션 개발부터 배포까지의 전체 과정을 **자동화**하는 것을 목표로 합니다.

*   **CI (Continuous Integration)**: 개발자가 코드를 Git과 같은 중앙 리포지토리에 푸시(Push)하면, **자동으로 빌드(Build)되고 테스트(Test)되는** 과정입니다.
    *   **우리의 맥락**: Spring Boot 코드를 빌드하여 JAR 파일을 만들고, 이 JAR 파일을 포함한 Docker 이미지를 빌드하는 단계.

*   **CD (Continuous Deployment)**: CI 단계가 성공적으로 완료되면, 빌드된 결과물(Docker 이미지)이 **자동으로 실제 운영 서버에 배포(Deploy)되는** 과정입니다.

*   **Jenkins vs GitHub Actions**:
    *   **Jenkins**: 전통적인 CI/CD 도구. 별도의 서버에 직접 설치하고 운영해야 하는 번거로움이 있지만, 플러그인이 매우 많아 확장성이 뛰어납니다.
    *   **GitHub Actions**: GitHub에 내장된 CI/CD 도구. 별도의 서버 없이 GitHub 리포지토리 내에서 `.yml` 설정 파일만으로 파이프라인을 정의할 수 있어, 시작하기 매우 간편하고 직관적입니다.

---

### ✅ 4. GitHub Actions로 AWS EC2에 자동 배포하기

*   GitHub 리포지토리의 `.github/workflows` 디렉토리에 **워크플로우(Workflow) YAML 파일**을 작성하여, 특정 이벤트(e.g., `main` 브랜치에 코드 푸시)가 발생했을 때 CI/CD 파이프라인이 자동으로 실행되도록 구성합니다.

#### ➕ 자동화 파이프라인의 단계별 흐름

1.  **트리거 (Trigger)**: `main` 브랜치에 코드가 푸시되면 워크플로우를 시작합니다.

2.  **[CI] 애플리케이션 빌드**:
    *   GitHub Actions 서버(Runner)에서 우리 프로젝트 코드를 가져옵니다.
    *   JDK를 설정하고, `./gradlew build` 명령어로 Spring Boot 애플리케이션을 빌드하여 JAR 파일을 생성합니다.

3.  **[CI] Docker 이미지 빌드 및 푸시**:
    *   **GitHub Secrets**에 안전하게 저장된 `DOCKER_USERNAME`과 `DOCKER_PASSWORD`를 사용하여 Docker Hub에 로그인합니다.
    *   `Dockerfile`을 사용하여 Docker 이미지를 빌드합니다.
    *   빌드된 이미지를 Docker Hub 리포지토리에 푸시합니다.

4.  **[CD] AWS EC2에 배포**:
    *   **GitHub Secrets**에 저장된 EC2 접속 정보(`EC2_HOST`, `EC2_USERNAME`, `EC2_SSH_KEY`)를 사용하여, GitHub Actions 서버가 EC2 인스턴스에 **SSH로 원격 접속**합니다.
    *   EC2 서버에서 다음 명령어들을 순차적으로 실행합니다.
        a. **Docker Hub 로그인**: EC2 서버도 Docker Hub에서 이미지를 받아와야 하므로 로그인이 필요합니다.
        b. **기존 컨테이너 중지 및 제거**: 이전에 실행 중이던 애플리케이션 컨테이너가 있다면, 충돌을 방지하기 위해 중지(`docker stop`)하고 제거(`docker rm`)합니다.
        c. **최신 이미지 풀(Pull)**: Docker Hub에서 방금 푸시한 최신 버전의 이미지를 내려받습니다.
        d. **새 컨테이너 실행**: 최신 이미지를 기반으로 새로운 컨테이너를 실행(`docker run`)하여 애플리케이션 배포를 완료합니다.

---

### 📌 요약

*   **Docker Hub**는 CI/CD 파이프라인에서 빌드된 이미지를 저장하고, 배포 서버가 이를 가져가는 **중간 다리** 역할을 합니다.
*   **EC2**에는 **Docker 환경**을 미리 구축해두어야, 원격 명령을 통해 컨테이너를 실행할 수 있습니다.
*   **GitHub Actions**는 Git 푸시와 같은 이벤트를 감지하여, **[빌드 → 테스트 → 이미지 생성 → Docker Hub 푸시 → EC2 접속 → 컨테이너 실행]**으로 이어지는 전체 **CI/CD 과정을 자동화**하는 강력한 도구입니다.
*   이 모든 자동화 과정에서 비밀번호나 SSH 키와 같은 민감한 정보는 **GitHub Secrets**를 통해 안전하게 관리해야 합니다.
