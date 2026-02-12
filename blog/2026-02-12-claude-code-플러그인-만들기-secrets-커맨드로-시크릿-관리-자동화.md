---
title: "Claude Code 플러그인 만들기: /secrets 커맨드로 시크릿 관리 자동화"
date: "2026-02-12"
tags: [claude-code, plugin]
---

# Claude Code 플러그인 만들기: /secrets 커맨드로 시크릿 관리 자동화

## 문제: 시크릿이 여기저기 흩어져 있다

개발 프로젝트가 6개를 넘어가면서, 각 프로젝트의 `.env` 파일 위치를 기억하는 게 고역이 됐다.

```
"contents-hub DB 비밀번호가 뭐였지?"
→ 폴더 들어가서 .env 열어보고...
"AWS 키는 어디 저장했더라?"
→ ssh-keys 폴더 뒤지고...
"프로덕션 Redis 주소는?"
→ docker-compose.prod.yml 찾아보고...
```

매번 이 삽질을 반복하다가, Claude Code 플러그인으로 해결했다.

## 해결: `/secrets` 커맨드 하나로 끝

```
/secrets                    → 전체 프로젝트 시크릿 목록
/secrets contents-hub       → contents-hub .env 실제 값 전체
/secrets anthropic          → Anthropic API Key 위치 + 값
/secrets aws                → AWS 자격증명 위치 + 내용
```

Claude Code에서 슬래시 커맨드 한 번이면 어떤 프로젝트의 어떤 시크릿이든 즉시 조회된다.

## 구현: 파일 3개면 충분하다

Claude Code 플러그인의 최소 구조는 놀랍도록 단순하다.

```
1password/
├── secrets-index.md              # 시크릿 인덱스 (수동 관리)
├── .claude-plugin/
│   └── plugin.json               # 플러그인 메타데이터
└── commands/
    └── secrets.md                # /secrets 커맨드 정의
```

### 1단계: plugin.json — 플러그인 등록

```json
{
  "name": "1password",
  "version": "1.0.0",
  "description": "시크릿 인덱스 조회",
  "author": { "name": "elon" }
}
```

이게 전부다. `.claude-plugin/plugin.json`만 있으면 Claude Code가 자동으로 플러그인을 인식한다.

### 2단계: secrets-index.md — 시크릿 지도

모든 프로젝트의 시크릿 위치를 하나의 마크다운 파일에 정리한다.

```markdown
## 프로젝트별 .env 위치

### contents-hub
- **파일**: ~/elon/ai/projects/contents_hub/backend/.env
- DB: `DATABASE_URL` (PostgreSQL)
- Redis: `REDIS_URL`
- SMTP: `SMTP_USER`, `SMTP_PASSWORD`
- Anthropic: `ANTHROPIC_API_KEY`
```

핵심 원칙: **변수명만 기록하고, 실제 값은 절대 넣지 않는다.** 값은 런타임에 `.env` 파일을 직접 읽어서 보여준다.

### 3단계: secrets.md — 커맨드 프롬프트

커맨드 정의는 마크다운 파일 하나다. YAML 프론트매터로 메타데이터를 설정하고, 본문에 AI가 따를 지침을 적는다.

```yaml
---
name: secrets
description: 시크릿 인덱스에서 비밀 정보 위치와 값을 조회합니다
argument-hint: "[프로젝트명 또는 키워드]"
allowed-tools:
  - Read
  - Glob
  - Grep
---
```

`allowed-tools`에 Read/Glob/Grep만 허용해서 **읽기 전용**으로 동작한다. 실수로 파일을 수정할 위험이 없다.

본문 프롬프트에서 3가지 동작 모드를 정의한다:

| 인자 | 동작 | 예시 |
|------|------|------|
| 없음 | 전체 프로젝트 목록 표시 | `/secrets` |
| 프로젝트명 | 해당 .env 파일 읽어서 전체 값 출력 | `/secrets contents-hub` |
| 키워드 | 인덱스 전체 검색 후 매칭 값 출력 | `/secrets smtp` |

하나의 커맨드에 인자 기반 분기 — 여러 커맨드를 나누는 것보다 사용자 경험이 훨씬 단순하다.

## 배운 것들

### 1. 플러그인 설치 3단계 전략

| 단계 | 방식 | 용도 |
|------|------|------|
| 개발 중 | 로컬 자동 인식 | `.claude-plugin/` 디렉토리만 있으면 자동 감지 |
| 안정화 | 로컬 마켓플레이스 | `known_marketplaces.json`에 등록 → 전역 사용 |
| 공유 | GitHub 마켓플레이스 | GitHub 레포로 배포 |

개발할 때는 해당 폴더에서 바로 쓰고, 검증되면 마켓플레이스로 승격한다.

### 2. 마켓플레이스 통합의 힘

처음에는 플러그인마다 별도 마켓플레이스를 만들었다. 1password, snapkin, disk-cleaner... 금방 복잡해졌다.

하나의 `my-plugins` 마켓플레이스로 통합하니 관리가 깔끔해졌다:

```json
{
  "name": "my-plugins",
  "plugins": [
    { "name": "1password", "source": "./1password" },
    { "name": "snapkin", "source": "./snapkin" },
    { "name": "disk-cleaner", "source": "./disk-cleaner" }
  ]
}
```

`source` 필드로 하위 디렉토리를 가리키면 끝. 새 플러그인은 배열에 한 줄만 추가하면 된다.

### 3. Command vs Skill — 언제 무엇을 쓸까

| 구분 | Command | Skill |
|------|---------|-------|
| 실행 | 사용자가 `/` 명령으로 직접 호출 | Claude가 대화 맥락에 따라 자동 적용 |
| 역할 | "무엇을 할지" (액션) | "어떻게 할지" (지식) |

`/secrets`는 사용자가 명시적으로 호출하는 **액션**이니까 Command가 맞다. "시크릿 관리 가이드라인"처럼 배경 지식이라면 Skill이 적절하다.

### 4. 리소스 배치의 원칙: 소속 범위

리소스(`scripts/`, `config/`, `references/`) 위치를 정하는 기준은 **누가 쓰느냐**다:

- 하나의 skill에서만 사용 → `skills/해당-skill/` 내부
- 여러 구성 요소가 공유 → 플러그인 루트
- 외부 서비스 연결 → `.mcp.json`

단순하지만, 이 원칙 하나로 "이 파일 어디에 둘까?" 고민이 사라진다.

## 실제 사용 모습

```
> /secrets contents-hub

📁 ~/elon/ai/projects/contents_hub/backend/.env

DATABASE_URL=postgresql://contents_hub:xxx@localhost:5432/contents_hub
REDIS_URL=redis://localhost:6379/0
ANTHROPIC_API_KEY=sk-ant-api03-xxx...
SMTP_USER=joomanba@gmail.com
...

🖥️ 운영 환경 (VM: elon.tpm.querypie.io)
- DB: pgvector-postgres-1:5432
- Redis: querypie-redis-1 (172.18.0.1:6379)
```

`.env` 파일 위치를 기억할 필요도, 터미널에서 `cat`으로 열 필요도 없다. Claude Code가 인덱스에서 경로를 찾고, 파일을 읽어서 보여준다.

## 마무리

Claude Code 플러그인 개발의 진입 장벽은 예상보다 훨씬 낮다.

- **파일 3개**로 완전한 플러그인이 만들어진다
- **코드 빌드 없이** 마크다운만으로 동작한다
- **allowed-tools**로 안전한 읽기 전용 플러그인을 만들 수 있다
- **마켓플레이스 통합**으로 여러 플러그인을 깔끔하게 관리할 수 있다

반복적으로 `.env` 파일을 찾아 헤매고 있다면, 30분 투자해서 플러그인 하나 만들어보자. 그 30분이 앞으로 수백 번의 삽질을 없애준다.
