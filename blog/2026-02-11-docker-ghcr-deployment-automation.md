---
title: "Docker + GHCR로 배포 자동화하기"
date: "2026-02-11"
tags: [docker, cicd, ghcr, devops]
---

# Docker + GHCR로 배포 자동화하기

> 1인 개발자가 사이드 프로젝트를 운영 서버에 배포하기까지의 여정

## 문제: "배포할 때마다 5분씩 기다려야 해"

사이드 프로젝트 [Contents Hub](https://github.com/elon-jang/contents-hub)를 운영하면서 배포가 점점 귀찮아졌다. 매번 rsync로 코드 올리고, VM에서 docker build 하면 5분. 작은 수정 하나에도 커피 한 잔 마시고 와야 했다.

```
# 기존 배포 절차 (고통)
rsync -avz ./backend/ server:/app/     # 30초
ssh server "docker build . -t app"      # 3-5분 (VM이 느려서...)
ssh server "docker-compose up -d"       # 10초
```

특히 VM 스펙이 낮아서 빌드가 느린 게 문제였다. 내 Mac에서는 30초면 되는 빌드가 VM에서는 5분.

## 해결: "로컬에서 빌드하고, 이미지만 올리자"

Docker 이미지 레지스트리를 쓰면 된다는 걸 알고 있었지만, "Docker Hub는 무료 플랜이 제한적이고, AWS ECR은 설정이 복잡하고..." 하면서 미뤄왔다.

그러다 **GHCR (GitHub Container Registry)**를 발견했다.

### GHCR을 선택한 이유

| 기능 | Docker Hub | GHCR |
|------|------------|------|
| Private 이미지 | 유료 ($5/월) | **무료** |
| Pull 제한 | 100회/6시간 | **없음** |
| GitHub Actions 연동 | 별도 설정 | **네이티브** |

1인 개발자에게 GHCR은 거의 완벽한 선택이었다.

## 첫 번째 삽질: ARM64 vs AMD64

Mac에서 빌드한 이미지를 VM에서 실행하니...

```
backend The requested image's platform (linux/arm64) does not match
the detected host platform (linux/amd64/v4)
```

M1 Mac은 ARM64, 대부분의 클라우드 VM은 AMD64. 플랫폼이 다르다.

**해결책**: `docker buildx`로 크로스 플랫폼 빌드

```bash
docker buildx build --platform linux/amd64 \
  -t ghcr.io/myuser/myapp:v1.0.0 \
  --push .
```

`--platform linux/amd64`를 붙이면 Mac에서도 Linux AMD64용 이미지를 빌드할 수 있다.

## 두 번째 삽질: GHCR 권한

`gh auth login`으로 GitHub 인증을 했는데 push가 안 된다.

```
denied: permission_denied: The token provided does not match expected scopes.
```

**원인**: GitHub CLI의 기본 인증에는 `write:packages` 권한이 없다.

**해결책**:
```bash
gh auth refresh -s write:packages
```

브라우저에서 추가 권한을 승인하면 된다.

## 세 번째 삽질: .env 변경이 안 먹혀

`.env`에서 Redis URL을 수정하고 `docker-compose restart`를 했는데, 여전히 옛날 설정을 사용한다.

**원인**: `restart`는 컨테이너를 재시작만 하고, 환경변수는 컨테이너 **생성 시점**에 주입된다.

**해결책**:
```bash
# restart 대신
docker-compose down && docker-compose up -d

# 또는
docker-compose up -d --force-recreate
```

## 완성된 배포 스크립트

삽질 끝에 완성한 원클릭 배포 스크립트:

```bash
#!/bin/bash
# scripts/deploy.sh
set -e

VERSION=${1:-latest}
IMAGE="ghcr.io/myuser/myapp"

echo "🧪 테스트 실행 중..."
pytest tests/ -v --tb=short -q

echo "🏗️ Docker 이미지 빌드 중 (linux/amd64)..."
docker buildx build --platform linux/amd64 \
    -t "$IMAGE:$VERSION" \
    -t "$IMAGE:latest" \
    --push .

echo "📦 VM에 배포 중..."
ssh server "
    docker pull $IMAGE:$VERSION
    docker-compose -f docker-compose.prod.yml up -d --force-recreate
"

echo "✅ 배포 완료! ($VERSION)"
```

이제 배포는:

```bash
./scripts/deploy.sh v1.0.1
```

**20초**면 끝난다. 테스트 → 빌드 → 푸시 → 배포가 자동으로.

## 핵심 교훈

### 1. 빌드는 로컬에서, 실행만 서버에서

VM에서 빌드하는 건 시간 낭비다. 로컬에서 빌드하고 이미지만 pull하면 된다.

### 2. 크로스 플랫폼은 필수 고려사항

M1/M2 Mac을 쓴다면 `--platform linux/amd64`를 습관처럼 붙이자.

### 3. 실패하면 다음 단계로 가지 마라

`set -e`로 스크립트가 실패 시 즉시 중단되게. 테스트 실패했는데 배포되는 사고를 막는다.

### 4. 버전 태그를 남겨라

`latest`만 쓰면 롤백이 어렵다. `v1.0.0`, `v1.0.1` 식으로 태그를 남기면 문제 발생 시:

```bash
docker pull ghcr.io/myuser/myapp:v1.0.0  # 이전 버전으로 롤백
```

## 다음 단계: GitHub Actions

지금은 `./scripts/deploy.sh`를 수동으로 실행하지만, 다음 목표는 GitHub Actions로 완전 자동화다:

```yaml
# .github/workflows/deploy.yml
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Build and push
        run: |
          docker buildx build --platform linux/amd64 \
            -t ghcr.io/${{ github.repository }}:${{ github.sha }} \
            --push .
```

`main`에 푸시하면 자동 배포. 꿈의 CI/CD다.

---

*1인 개발자도 DevOps 할 수 있다. 복잡할 것 없다. 한 단계씩.*
