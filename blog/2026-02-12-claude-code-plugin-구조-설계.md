---
title: "Claude Code Plugin 구조 설계"
date: "2026-02-12"
tags: [claude-code, plugin, architecture]
---

# Claude Code Plugin 구조 설계

disk-cleaner 프로젝트를 Skill에서 Plugin으로 전환하면서 배운 설계 원칙을 정리한다.

## Skill vs Plugin, 언제 무엇을 선택하나

Claude Code에서 기능을 패키징하는 방식은 크게 두 가지다:

| 타입 | 트리거 | 역할 | 예시 |
|------|--------|------|------|
| **Type B: Skill** | Claude가 대화 맥락에 따라 자동 적용 | "어떻게 할지" — 도메인 지식 | 코드 리뷰 가이드라인 |
| **Type A: Plugin** | 사용자가 `/command`로 직접 호출 | "무엇을 할지" — 액션 진입점 | `/disk-cleaner:clean` |

**선택 기준은 단순하다**: 사용자가 명시적으로 실행해야 하면 Plugin, Claude가 알아서 참고하면 되면 Skill.

disk-cleaner는 "디스크 정리해줘"라고 할 때 스크립트를 실행해야 하므로 Plugin이 맞다. Skill로 만들면 slash command가 없어서 사용자가 직접 호출할 수 없다.

## Command + Skill 분리 패턴

Plugin의 핵심은 Command와 Skill의 분리다:

```
commands/clean.md       → "무엇을 할지" (스크립트 실행, 결과 요약)
skills/disk-cleaner/    → "어떻게 할지" (cleaner 목록, 안전 장치, 설정)
```

Command는 가볍게 — 실행 방법과 결과 안내만 담는다. Skill은 Claude가 Command 실행 시 **자동으로 참조**하는 배경 지식이다. 이 분리 덕분에:

- Command를 읽는 비용이 낮아진다 (프롬프트 토큰 절약)
- Skill은 대화 맥락에서도 자동 활성화된다
- 각각 독립적으로 업데이트할 수 있다

## 표준 디렉토리 구조

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json                # 플러그인 매니페스트
├── commands/
│   └── clean.md                   # /plugin:command 진입점
├── skills/
│   └── domain-knowledge/
│       └── SKILL.md               # 자동 참조 도메인 지식
├── scripts/                       # 실행 로직 통일
├── config/                        # 설정 파일
├── output/                        # 생성물 (.gitignore)
└── README.md
```

배치 원칙: **소속 범위로 결정한다**.

- 하나의 skill에서만 쓰면 → `skills/해당-skill/` 내부
- 여러 구성 요소가 공유하면 → 플러그인 루트
- 실행 로직은 항상 → `scripts/`

## 마이그레이션 실전 기록

### Before (Type B: Skill)

```
disk-cleaner/
├── SKILL.md                       # 루트에 단일 파일
├── disk-cleaner                   # 실행 스크립트 (루트)
├── install.sh                     # 루트
└── disk-cleaner.conf.example      # 루트
```

### After (Type A: Plugin)

```
disk-cleaner/
├── .claude-plugin/plugin.json     # 매니페스트 추가
├── commands/clean.md              # Command 분리
├── skills/disk-cleaner/SKILL.md   # Skill 분리
├── scripts/disk-cleaner           # scripts/로 이동
├── scripts/install.sh             # scripts/로 이동
├── config/disk-cleaner.conf.example
└── output/REPORT-*.md             # .gitignore
```

### 마이그레이션 체크리스트

1. `.claude-plugin/plugin.json` 생성 (name, version, description)
2. SKILL.md를 Command + Skill로 분리
3. 실행 스크립트를 `scripts/`로 이동
4. 설정 파일을 `config/`로 이동
5. 생성물을 `output/`으로 이동 + `.gitignore`
6. `settings.json` → `settings.local.json`으로 통합
7. 내부 경로 참조 업데이트 (install.sh, SKILL.md, README.md)
8. 로컬 마켓플레이스에 등록

## 로컬 마켓플레이스로 전역 사용

Plugin을 만들면 해당 디렉토리에서만 인식된다. 다른 프로젝트에서도 쓰려면 **로컬 마켓플레이스**에 등록:

```json
// ~/elon/ai/projects/.claude-plugin/marketplace.json
{
  "plugins": [
    {
      "name": "disk-cleaner",
      "source": "./disk-cleaner",
      "version": "1.0.0"
    }
  ]
}
```

등록 후 Claude Code 재시작 → `/plugin` → install. 이제 어떤 프로젝트에서든 `/disk-cleaner:clean`으로 호출 가능.

## 핵심 교훈

1. **Skill과 Plugin은 용도가 다르다** — "자동 참조"와 "명시적 실행"의 차이
2. **Command는 가볍게, Skill은 풍부하게** — 분리하면 토큰도 절약되고 유지보수도 쉽다
3. **디렉토리 구조는 컨벤션을 따르라** — `scripts/`, `config/`, `output/` 같은 표준 위치가 있으면 고민할 필요 없다
4. **마켓플레이스 등록은 간단하다** — JSON 한 줄이면 전역 사용 가능
