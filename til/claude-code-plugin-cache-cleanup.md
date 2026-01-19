---
id: "20260119-a1b2c3"
title: "Claude Code 플러그인 캐시 삭제로 완전 제거하기"
category: "til"
tags: ["claude-code", "plugin", "troubleshooting", "cache"]
created: "2026-01-19T00:00:00+09:00"
source: "/Users/elon/elon/ai/PLUGIN_TROUBLESHOOTING.md"
confidence: 3
connections: []
review_count: 0
last_reviewed: null
---

# Claude Code 플러그인 캐시 삭제로 완전 제거하기

## Summary

Claude Code 플러그인 삭제 후에도 suggestion에 계속 나타나는 문제 해결법. `/plugin uninstall`은 레지스트리만 제거하고 캐시는 남기므로 수동으로 캐시 경로들을 삭제해야 함.

## Key Points

- 캐시 위치: `plugins/cache`, `marketplaces`, `skills`
- `find` 명령어로 관련 폴더 찾기: `find ~/.claude -name "*<plugin-name>*" -type d`
- 모든 캐시 삭제 후 Claude Code 재시작 필요

## Questions

### Q1: /plugin uninstall 명령어로 플러그인을 제거했는데 왜 계속 suggestion에 나타나는가?

**A**: `/plugin uninstall`은 `installed_plugins.json` 레지스트리에서만 제거하고, `plugins/cache`, `marketplaces`, `skills` 폴더의 캐시 파일들은 남기기 때문.

### Q2: Claude Code 플러그인 캐시가 저장되는 3가지 주요 경로는?

**A**: `~/.claude/plugins/cache`, `~/.claude/plugins/marketplaces/.../plugins`, `~/.claude/skills`

### Q3: 삭제된 플러그인의 잔여 캐시를 찾는 명령어는?

**A**: `find ~/.claude -name "*<plugin-name>*" -type d 2>/dev/null`

## My Understanding

<!-- Add your own understanding here -->
