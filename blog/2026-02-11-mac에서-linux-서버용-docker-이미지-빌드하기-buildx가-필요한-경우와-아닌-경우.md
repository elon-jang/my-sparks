---
title: "Mac에서 Linux 서버용 Docker 이미지 빌드하기: buildx가 필요한 경우와 아닌 경우"
date: "2026-02-11"
tags: [docker, deployment, cross-platform, devops]
---

# Mac에서 Linux 서버용 Docker 이미지 빌드하기: buildx가 필요한 경우와 아닌 경우

## 문제 상황

Mac M1/M2(ARM64)에서 개발하고, 운영 서버(Linux AMD64)에 Docker로 배포한다. 아키텍처가 다르니까 이미지를 그냥 빌드하면 서버에서 안 돌아간다.

```
exec format error
```

이 에러를 만나본 적 있다면, 이 글이 도움이 될 것이다.

## 두 가지 해결법

### 방법 1: `docker buildx` (QEMU 에뮬레이션)

Mac 위에서 가상으로 AMD64 환경을 만들어 빌드한다.

```bash
docker buildx build --platform linux/amd64 -t my-image:latest --push .
```

**작동 원리:**
1. Docker가 BuildKit 컨테이너를 띄움
2. QEMU로 AMD64 CPU를 에뮬레이션
3. 그 안에서 `apt-get install`, `pip install`, 바이너리 다운로드 등 실행
4. 결과물이 AMD64용 이미지

### 방법 2: 빌드 분리 (로컬 빌드 + 결과물 복사)

로컬에서 빌드를 완료하고, 결과물만 이미지에 넣는다.

```bash
# 로컬에서 빌드
npm run build

# 결과물만 이미지에 복사
docker build -f Dockerfile.prebuilt .
```

```dockerfile
# Dockerfile.prebuilt
FROM nginx:alpine
COPY build/ /usr/share/nginx/html
```

## 언제 어떤 방법을 쓸까?

핵심 판단 기준은 하나다: **컨테이너 안에서 네이티브 바이너리 설치/실행이 필요한가?**

### buildx가 필요한 경우 (Contents Hub 백엔드)

```dockerfile
FROM python:3.11-slim

# 시스템 라이브러리 15개 설치 (amd64용)
RUN apt-get install -y libnss3 libgbm1 libasound2 ...

# Python C 확장 컴파일 (amd64용 바이너리)
RUN pip install lxml asyncpg playwright

# amd64용 Chromium 다운로드
RUN playwright install chromium

# 컨테이너 안에서 Python 프로세스 실행
CMD ["uvicorn", "app.main:app"]
```

여기서 일어나는 일:
- `apt-get install` → amd64용 `.so` 라이브러리 설치
- `pip install lxml` → C 코드를 amd64로 컴파일
- `playwright install chromium` → amd64 Chromium 바이너리 다운로드
- `uvicorn` → amd64에서 Python 인터프리터 실행

이것들은 **타겟 아키텍처 위에서 직접 실행**되어야 한다. 로컬 Mac에서 미리 빌드해서 복사할 수 없다. ARM용 `.so` 파일은 AMD64에서 못 쓴다.

### buildx가 필요 없는 경우 (프론트엔드 정적 빌드)

```dockerfile
FROM nginx:alpine
COPY deploy/nginx/frontend.conf /etc/nginx/conf.d/default.conf
COPY build/ /usr/share/nginx/html
EXPOSE 80
```

여기서 일어나는 일:
- `nginx:alpine` → 멀티 플랫폼 이미지 (자동으로 amd64 선택)
- `COPY build/` → HTML, JS, CSS 파일 복사
- nginx가 정적 파일 서빙

`npm run build`의 결과물은 HTML/JS/CSS — **아키텍처와 무관한 정적 파일**이다. Mac에서 빌드하든 Linux에서 빌드하든 결과물은 동일하다.

## 실제 사례 비교

| | 프론트엔드 (빌드 분리) | 백엔드 (buildx) |
|---|---|---|
| **프로젝트** | econipass 프론트 | Contents Hub 백엔드 |
| **빌드 결과물** | 정적 HTML/JS/CSS | 없음 (런타임 실행) |
| **네이티브 의존성** | 없음 | Playwright + Chromium + C 확장 |
| **QEMU 필요** | 불필요 | 필수 |
| **빌드 시간** | 빠름 (로컬 빌드) | 느림 (에뮬레이션 오버헤드) |

econipass 프론트엔드는 처음에 `docker buildx`로 빌드했다가 실패했다:

```
env: can't execute 'node': Exec format error
```

QEMU가 `node` 바이너리를 에뮬레이션하는데, `style-dictionary` 패키지가 네이티브 node를 직접 호출하면서 터진 것. 빌드 분리 패턴으로 바꾸니 깔끔하게 해결됐다.

## 판단 플로우차트

```
컨테이너 안에서 뭘 하나?
│
├─ 정적 파일만 서빙 (nginx, S3)
│  → 빌드 분리: 로컬 빌드 + COPY
│
├─ 런타임 실행 (Python, Node, Java)
│  │
│  ├─ 네이티브 C 확장 있음 (lxml, bcrypt, sharp)
│  │  → buildx 필수
│  │
│  └─ Pure JS/Python만 사용
│     → buildx 권장 (안전), 빌드 분리도 가능
│
└─ 시스템 라이브러리 필요 (apt-get install)
   → buildx 필수
```

## buildx 사용 시 팁

### 1. 빌더 컨테이너는 배포 후 정리

```bash
# 빌더 컨테이너 확인
docker ps --filter "name=buildx"

# 평소에는 정리 (231MB 절약)
docker stop buildx_buildkit_mybuilder0
docker rm buildx_buildkit_mybuilder0

# 다음 buildx 실행 시 자동 재생성됨
```

### 2. 캐시 문제 주의

`docker buildx`는 별도 BuildKit 컨테이너에서 빌드하므로 캐시가 파일 변경을 못 잡을 수 있다. 코드 변경이 반영 안 되면:

```bash
docker buildx build --no-cache --platform linux/amd64 --push .
```

### 3. 배포 스크립트에 플랫폼 명시

```bash
# deploy.sh
docker buildx build \
  --platform linux/amd64 \
  -t ghcr.io/my-org/my-app:$VERSION \
  -t ghcr.io/my-org/my-app:latest \
  --push .
```

`--platform`을 빠뜨리면 Mac ARM64 이미지가 빌드되어 서버에서 `exec format error` 발생.

## 정리

**한 줄 요약**: 컨테이너 안에서 네이티브 바이너리를 설치하거나 실행해야 하면 `buildx`, 정적 파일만 복사하면 빌드 분리.

| 상황 | 방법 |
|------|------|
| 프론트엔드 → nginx | 빌드 분리 |
| Python + C 확장 | buildx |
| Node.js + native addon | buildx |
| Java JAR | 빌드 분리 가능 |
| Go 바이너리 | `GOARCH=amd64` 크로스 컴파일 |

크로스 플랫폼 빌드에서 문제가 생기면, "이 작업이 타겟 아키텍처에서 실행되어야 하는가?"를 먼저 질문하자. 답이 "예"면 buildx, "아니오"면 빌드 분리.
