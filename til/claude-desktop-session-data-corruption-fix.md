---
id: "20260121-cld001"
title: "Claude Desktop 세션 데이터 손상 시 완전 초기화로 해결하기"
category: "til"
tags: ["claude-desktop", "troubleshooting", "macos", "mcp"]
created: "2026-01-21T15:21:00+09:00"
source: "경험"
blog_link: "blog/2026-01-21-claude-desktop-먹통-해결-삽질-기록.md"
confidence: 3
connections: ["20260121-mcp001"]
review_count: 0
last_reviewed: null
---

# Claude Desktop 세션 데이터 손상 시 완전 초기화로 해결하기

> **Blog**: [Claude Desktop이 갑자기 먹통이 됐을 때 - 삽질 3시간의 기록](../blog/2026-01-21-claude-desktop-먹통-해결-삽질-기록.md)

## Summary

Claude Desktop에서 메시지를 보내자마자 대화창이 초기화면으로 돌아가는 문제는 세션 데이터 손상이 원인입니다. 캐시나 일부 파일 삭제로는 해결되지 않으며, `claude_desktop_config.json`만 백업한 후 전체 초기화해야 합니다.

## Key Points

- **증상**: 웹(claude.ai)은 되는데 앱만 안 됨 → 앱 내부 데이터 문제
- **로그 확인**: `~/Library/Logs/Claude/claude.ai-web.log`에서 `Internal server error`, `Invalid authorization` 확인
- **부분 삭제는 미련**: Cache, Session Storage, Local Storage 개별 삭제로는 해결 안 됨
- **해결법**: 설정 파일(`claude_desktop_config.json`)만 백업 후 전체 삭제
- **MCP 설정 보존**: `claude_desktop_config.json`에 MCP 서버 설정이 저장되어 있으므로 반드시 백업

## Questions

### Q1: Claude Desktop에서 대화가 계속 초기화될 때 로그에서 확인해야 할 에러는?

**A**: `~/Library/Logs/Claude/claude.ai-web.log`에서 `Internal server error`와 `Invalid authorization` 에러를 확인합니다.

### Q2: Claude Desktop 완전 초기화 명령어는?

**A**:
```bash
pkill -f "Claude"
cp ~/Library/Application\ Support/Claude/claude_desktop_config.json /tmp/
rm -rf ~/Library/Application\ Support/Claude/*
cp /tmp/claude_desktop_config.json ~/Library/Application\ Support/Claude/
open -a "Claude"
```

### Q3: 웹은 되는데 앱만 안 될 때 문제의 원인은?

**A**: 앱 내부 세션 데이터(Session Storage, Local Storage, IndexedDB, Cookies 등)가 손상되어 "로그인은 됐는데 API 호출이 안 되는" 상태입니다.

## My Understanding

<!-- Add your own understanding here -->
