Nginx 환경에서 CI/CD(지속적 통합 및 배포)를 구축하여 자동 배포를 구현하는 방법은 크게 **GitHub Actions**나 **Jenkins** 같은 CI 도구와 **배포 대상 서버**를 연결하는 흐름으로 구성됩니다.

가장 대중적이고 설정이 간편한 **GitHub Actions와 Docker**를 이용한 자동 배포 프로세스를 중심으로 설명해 드릴게요.

---

## 1. 자동 배포 기본 아키텍처

전형적인 흐름은 다음과 같습니다:

1. **Push**: 개발자가 코드를 GitHub 메인 브랜치에 올립니다.
2. **Build**: GitHub Actions가 코드를 빌드하고 Docker 이미지를 만듭니다.
3. **Push Image**: 생성된 이미지를 Docker Hub나 배포 레지스트리에 저장합니다.
4. **Deploy**: 운영 서버에 접속하여 새 이미지를 내려받고 Nginx 컨테이너를 재시작합니다.

---

## 2. 단계별 구현 방법

### Step 1: 프로젝트 내 Dockerfile 작성

Nginx는 정적 파일(HTML, JS, CSS)을 서빙하거나 프록시 역할을 하므로, 이를 포함하는 이미지를 정의해야 합니다.

```dockerfile
# Dockerfile
FROM nginx:latest
# 빌드된 정적 파일들을 nginx 경로로 복사
COPY ./dist /usr/share/nginx/html
# 사용자 정의 nginx 설정이 있다면 복사
COPY ./nginx/default.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

```

### Step 2: GitHub Actions 워크플로우 설정

`.github/workflows/deploy.yml` 파일을 생성하여 자동화 과정을 정의합니다.

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ "main" ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      # 1. Docker 이미지 빌드 및 푸시
      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push
        run: |
          docker build -t your-id/nginx-app:latest .
          docker push your-id/nginx-app:latest

      # 2. SSH로 서버 접속 후 배포 (Appleboy SSH Action 사용)
      - name: Deploy to Server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_KEY }}
          script: |
            docker pull your-id/nginx-app:latest
            docker stop nginx-container || true
            docker rm nginx-container || true
            docker run -d --name nginx-container -p 80:80 your-id/nginx-app:latest

```

---

## 3. 무중단 배포를 위한 팁 (Blue-Green)

단순히 컨테이너를 끄고 켜면 배포 중에 서비스가 잠시 중단됩니다. 이를 방지하려면 다음 방법을 고려하세요.

* **Nginx Upstream 활용**: 서버 내에 두 개의 컨테이너(Blue, Green)를 띄우고, Nginx가 가리키는 포트만 변경한 뒤 `nginx -s reload`를 실행합니다.
* **Docker Compose**: `docker-compose up -d --build`를 통해 서비스 중단을 최소화하며 컨테이너를 갱신할 수 있습니다.

---

## 4. 보안 및 체크리스트

* **Secrets 관리**: 서버 IP, SSH 키, DB 비밀번호 등은 반드시 GitHub Repository Settings > Secrets에 저장하세요.
* **Health Check**: 배포 후 새 컨테이너가 정상 작동하는지 확인하는 스크립트를 워크플로우 마지막에 추가하는 것이 좋습니다.

현재 사용하시는 **프로그래밍 언어(React, Vue, Spring 등)**나 **서버 환경(AWS, 일반 VPS 등)**을 알려주시면, 그에 맞춘 더 구체적인 설정 코드를 작성해 드릴 수 있습니다.

**어떤 프로젝트를 배포하려고 하시나요?**
