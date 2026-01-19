---
id: "20260119-d4e5f6"
title: "Claude Code 명령어 네이밍 리팩토링 - 96개 명령어 정리법"
category: "skills"
tags: ["claude-code", "command", "naming", "convention", "productivity"]
created: "2026-01-19T00:00:00+09:00"
source: "경험 분석"
blog_link: "blog/2026-01-19-claude-code-명령어-네이밍-리팩토링.md"
confidence: 3
connections: []
review_count: 0
last_reviewed: null
---

# Claude Code 명령어 네이밍 리팩토링 - 96개 명령어 정리법

## Summary

Claude Code에 96개 이상의 명령어가 있을 때, 제각각인 네이밍 스타일(케밥, 네임스페이스, 심플, 중복) 때문에 매번 help를 쳐야 하는 문제를 `[Category]:[Verb]` 공식으로 해결합니다. IDE 자동완성과 궁합이 좋고, 예측 가능하며, 타이핑이 줄어듭니다.

## Key Points

- **현재 문제**: 4가지 스타일 혼재 (케밥: `/shortcut-add`, 네임스페이스: `/sc:build`, 심플: `/docx`, 중복: `/session-wrap:wrap`)
- **해결 공식**: `/[Category]:[Verb]` 구조 통일
- **장점 3가지**: 예측 가능(접두어만 쳐도 관련 명령어 조회), 타이핑 절약(Tab 키 활용), 직관적(무엇을 어떻게)
- **표준 동사**: `add`, `list`, `find`, `rm`, `run`
- **Before→After 예시**:
  - `/shortcut-add` → `/shortcut:add`
  - `/docx` → `/doc:word`
  - `/frontend-design` → `/dev:ui`
  - `/sc:troubleshoot` → `/sc:fix`

## Questions

### Q1: Claude Code에서 현재 혼재된 명령어 네이밍 스타일 4가지는?

**A**: 케밥 스타일(`/shortcut-add`), 네임스페이스 파(`/sc:build`), 심플 파(`/docx`), 투머치 파(`/session-wrap:wrap` - namespace와 command 중복)

### Q2: `/[Category]:[Verb]` 공식의 장점 3가지는?

**A**: 1) 예측 가능 - 접두어만 쳐도 카테고리 내 모든 명령어 조회, 2) 타이핑 절약 - Tab 키로 실행 가능, 3) 직관적 - 무엇을(Category) 어떻게(Verb) 할지 명확

### Q3: `/frontend-design`을 새 네이밍 규칙에 맞게 리팩토링하면?

**A**: `/dev:ui` - 개발(dev) 카테고리 아래 UI 관련 기능으로 분류

## My Understanding

<!-- Add your own understanding here -->
