---
id: "20260119-d4e5f6"
title: "CLI 명령어 네이밍 컨벤션 - Scope:Action 구조"
category: "skills"
tags: ["claude-code", "command", "naming"]
created: "2026-01-19T00:00:00+09:00"
source: "경험 분석"
confidence: 3
connections: []
review_count: 0
last_reviewed: null
---

# CLI 명령어 네이밍 컨벤션 - Scope:Action 구조

## Summary

96개 이상의 CLI 명령어를 관리할 때 `[Scope]:[Action]` 형태의 네임스페이스 구조를 도입하면 IDE 자동완성 효율이 높아지고 사용자가 명령어를 예측할 수 있게 됩니다.

## Key Points

- **네이밍 구조**: `[Scope]:[Action]` 형태 권장 (예: `/shortcut:add`, `/doc:word`)
- **표준 Prefix**: `sc:`(Core), `doc:`(Document), `util:`(Utilities), `dev:`(Development), `sys:`(System)
- **표준 동사**: `add`(생성), `list`(조회), `rm`(삭제), `run`(실행), `find`(검색)
- **피해야 할 패턴**: `/session-wrap:wrap` 처럼 namespace와 command가 중복되는 네이밍
- **별칭(Alias)**: 자주 쓰는 명령어는 짧은 별칭 허용 (예: `/docx` → `/doc:word`)

## Questions

### Q1: CLI 명령어가 100개 가까이 될 때 추천하는 네이밍 구조는?

**A**: `[Scope]:[Action]` 형태의 네임스페이스 구조. 예: `/shortcut:add`, `/doc:word`. IDE 자동완성과 궁합이 좋고 관련 기능을 그룹으로 조회할 수 있음.

### Q2: 명령어 동사(Verb) 통일 시 권장하는 표준 동사 5가지는?

**A**: `add`(생성), `list`(조회), `rm`(삭제), `run`(실행), `find`(검색). 사용자가 명령어를 외우지 않아도 유추할 수 있게 됨.

### Q3: /session-wrap:wrap 같은 네이밍이 나쁜 이유는?

**A**: namespace와 command가 중복되어 불필요하게 길어짐. `/session:wrap` 또는 `/session:end`로 개선해야 함.

## My Understanding

<!-- Add your own understanding here -->
