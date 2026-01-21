---
id: "20260121-mcp001"
title: "Claude Desktop MCP 서버 연동 트러블슈팅"
category: "til"
tags: ["claude-desktop", "mcp", "python", "pyenv", "docker", "troubleshooting"]
created: "2026-01-21T16:00:00+09:00"
source: "경험"
blog_link: "blog/2026-01-21-claude-desktop-mcp-서버-연동-삽질기.md"
confidence: 3
connections: []
review_count: 0
last_reviewed: null
---

# Claude Desktop MCP 서버 연동 트러블슈팅

> **Blog**: [Claude Desktop MCP 서버 연동 삽질기 - 6시간의 대장정](../blog/2026-01-21-claude-desktop-mcp-서버-연동-삽질기.md)

## Summary

Docker 기반 MCP 서버가 Claude Desktop에서 작동하지 않을 때, Python 직접 실행 방식으로 해결할 수 있습니다. 핵심은 `/bin/bash -c`로 명령어 체인을 실행하고, pyenv Python 절대 경로를 사용하는 것입니다.

## Key Points

- **Docker MCP 서버 연결 실패 시**: Python 직접 실행 방식으로 전환
- **cwd 옵션 미작동**: `cd` 명령어로 작업 디렉토리 직접 이동
- **시스템 Python 문제**: pyenv Python 절대 경로 사용 (의존성 설치된 환경)
- **환경 변수 설정**: 인라인으로 환경 변수 지정 (`DATA_DIR=... python3 ...`)
- **로그 확인 위치**: `~/Library/Logs/Claude/` (mcp-server-*.log, main.log, claude.ai-web.log)

## Questions

### Q1: Claude Desktop에서 Docker 기반 MCP 서버가 작동하지 않을 때 대안은?

**A**: Python 직접 실행 방식으로 전환합니다. `/bin/bash -c`로 명령어 체인을 실행하고, pyenv Python 절대 경로를 사용합니다.

### Q2: MCP 설정에서 cwd 옵션이 작동하지 않을 때 해결 방법은?

**A**: bash -c 안에서 `cd /path/to/project &&` 형태로 작업 디렉토리를 직접 이동합니다.

### Q3: Claude가 MCP 도구를 실제로 호출했는지 확인하는 방법은?

**A**: `grep "tools/call" ~/Library/Logs/Claude/mcp-server-*.log` 명령어로 MCP 로그에서 tools/call 기록을 확인합니다. 기록이 없으면 Claude가 도구를 호출하지 않고 가짜 응답을 생성한 것입니다.

## My Understanding

<!-- Add your own understanding here -->
