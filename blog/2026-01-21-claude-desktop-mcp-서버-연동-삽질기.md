---
title: "Claude Desktop MCP 서버 연동 삽질기 - 6시간의 대장정"
date: "2026-01-21"
tags: [claude-desktop, mcp, python, pyenv, docker, troubleshooting]
---

## Claude Desktop MCP 서버 연동 삽질기 - 6시간의 대장정

### TL;DR

Docker 기반 MCP 서버가 Claude Desktop에서 작동하지 않아, Python 직접 실행 방식으로 변경하여 해결했습니다. 핵심은 올바른 Python 경로와 작업 디렉토리 설정이었습니다.

---

### 1. 증상: 세 가지 문제

#### 문제 1: 대화가 초기화됨

Claude Desktop에서 아무 말이나 입력하면 대화창이 휙 하고 초기화면으로 돌아감.

**원인**: 앱 내부 세션 데이터 손상

```
[COMPLETION] Request failed Error: Internal server error
[REACT_QUERY_CLIENT] QueryClient error: Error: Invalid authorization
```

**해결**:
```bash
# 설정 파일 백업 후 전체 삭제
cp ~/Library/Application\ Support/Claude/claude_desktop_config.json /tmp/
rm -rf ~/Library/Application\ Support/Claude/*
cp /tmp/claude_desktop_config.json ~/Library/Application\ Support/Claude/
```

---

#### 문제 2: MCP 도구가 호출되지 않음

Claude가 "영수증 대상자 94명입니다"라고 답하지만, 실제 데이터는 3명이라고 표시.

**원인**: Claude가 MCP 도구를 실제로 호출하지 않고 도구 설명만 보고 가상의 응답을 생성

**증거**: MCP 로그에 tools/call 기록 없음
```bash
tail ~/Library/Logs/Claude/mcp-server-oikos-receipt.log | grep "tools/call"
# 결과: 없음
```

**비교**: Bitly MCP는 실제 단축 URL(https://bit.ly/xxx)을 생성함 → MCP 호출 자체는 작동

---

#### 문제 3: Docker MCP 서버 연결 실패

```
[MCP] Could not connect to MCP server oikos-receipt
Error: reply was never sent
```

**원인**: Claude Desktop이 Docker 기반 MCP 서버와 통신하는 데 문제 발생

---

### 2. 해결 과정

#### 시도 1: Docker → 실패 ❌

```json
{
  "command": "docker",
  "args": ["run", "-i", "--rm", "-v", "/Users/elon/donation_receipts:/data", "oikos-receipt:latest"]
}
```
- Docker 컨테이너 자체는 정상 작동 (터미널에서 테스트 성공)
- Claude Desktop에서만 연결 실패

#### 시도 2: Python 직접 실행 (cwd 옵션) → 실패 ❌

```json
{
  "command": "python3",
  "args": ["-m", "mcp_server.server"],
  "cwd": "/Users/elon/.../tax_return"
}
```
- cwd 옵션이 적용되지 않음
- `ModuleNotFoundError: No module named 'mcp_server'`

#### 시도 3: bash -c로 cd 명령 포함 → 실패 ❌

```json
{
  "command": "/bin/bash",
  "args": ["-c", "cd /path/to/project && python3 -m mcp_server.server"]
}
```
- `ModuleNotFoundError: No module named 'fastmcp'`
- Claude Desktop이 시스템 Python 사용 (fastmcp 미설치)

#### 시도 4: pyenv Python 절대 경로 사용 → 성공 ✅

```json
{
  "command": "/bin/bash",
  "args": [
    "-c",
    "cd /Users/elon/.../tax_return && DATA_DIR=/Users/elon/donation_receipts /Users/elon/.pyenv/versions/myenv/bin/python3 -m mcp_server.server"
  ]
}
```

---

### 3. 최종 작동 설정

```json
{
  "mcpServers": {
    "oikos-receipt": {
      "command": "/bin/bash",
      "args": [
        "-c",
        "cd /Users/elon/elon/ai/projects/church/oikos/tax_return && DATA_DIR=/Users/elon/donation_receipts /Users/elon/.pyenv/versions/myenv/bin/python3 -m mcp_server.server"
      ]
    }
  }
}
```

**핵심 포인트**:
1. `/bin/bash -c`로 명령어 체인 실행
2. `cd`로 작업 디렉토리 이동 (cwd 옵션 대신)
3. 환경 변수(DATA_DIR) 인라인 설정
4. pyenv Python 절대 경로 사용 (의존성 설치된 환경)

---

### 4. 개선이 필요한 부분

#### 4.1 Claude Desktop 측

| 문제 | 개선 제안 |
|------|----------|
| cwd 옵션 미작동 | 문서화 또는 버그 수정 필요 |
| Docker MCP 불안정 | Docker 기반 MCP 지원 개선 |
| 세션 데이터 손상 | 자동 복구 메커니즘 필요 |
| MCP 미호출 시 가짜 응답 | 도구 호출 실패 시 명확한 에러 표시 |

#### 4.2 MCP 서버 측 (oikos-receipt)

```python
# 현재: 도구 이름이 불명확
def tool_list_recipients(): ...

# 개선: 명확한 도구 이름
def list_donation_recipients(): ...
```

#### 4.3 설치 스크립트 개선

```bash
# 현재: Docker만 지원
docker build -t oikos-receipt .

# 개선: Python 직접 실행 옵션 추가
# install.sh에 추가
if [ "$USE_DOCKER" = "false" ]; then
  pip install -r requirements.txt
  # Python 경로를 자동 감지하여 설정
fi
```

#### 4.4 문서화 필요 사항

```markdown
## Claude Desktop MCP 설정 가이드

### 방법 1: Docker (권장하지 않음)
- Claude Desktop과 호환성 문제 있음

### 방법 2: Python 직접 실행 (권장)
1. 의존성 설치: `pip install -r requirements.txt`
2. Python 경로 확인: `which python3`
3. 설정 파일에 절대 경로 사용
```

---

### 5. 디버깅 치트시트

#### 로그 파일 위치

```bash
# MCP 서버 로그
~/Library/Logs/Claude/mcp-server-oikos-receipt.log

# Claude Desktop 메인 로그
~/Library/Logs/Claude/main.log

# 웹 뷰 로그 (API 에러)
~/Library/Logs/Claude/claude.ai-web.log
```

#### 유용한 명령어

```bash
# MCP 도구 호출 확인
grep "tools/call" ~/Library/Logs/Claude/mcp-server-*.log

# 에러 확인
grep -i "error" ~/Library/Logs/Claude/claude.ai-web.log | tail -20

# Claude Desktop 완전 초기화
pkill -f "Claude"
cp ~/Library/Application\ Support/Claude/claude_desktop_config.json /tmp/
rm -rf ~/Library/Application\ Support/Claude/*
cp /tmp/claude_desktop_config.json ~/Library/Application\ Support/Claude/
open -a "Claude"
```

#### MCP 서버 직접 테스트

```bash
echo '{"jsonrpc":"2.0","method":"initialize",...}
{"jsonrpc":"2.0","method":"tools/call","params":{"name":"list_donation_recipients","arguments":{}},"id":1}' | \
python3 -m mcp_server.server
```

---

### 6. 교훈

1. **Docker가 항상 정답은 아니다** - 호환성 문제가 있을 수 있음
2. **로그는 친구다** - `~/Library/Logs/Claude/`를 자주 확인
3. **Claude의 응답을 맹신하지 말자** - MCP 미호출 시 가짜 응답 생성 가능
4. **절대 경로를 사용하자** - 환경 변수, Python 경로 모두
5. **과감히 밀어버리자** - 부분 수정보다 전체 초기화가 빠를 때도 있음

---

### 7. 결론

6시간의 삽질 끝에 MCP 서버가 드디어 작동합니다.

```
사용자: 영수증 대상자 목록 보여줘
Claude: 총 94명의 대상자가 있습니다. 총 금액은 ₩50,820,000입니다.
```

진짜 데이터가 나왔습니다!

---

이 글이 Claude Desktop MCP 연동에 어려움을 겪는 분들께 도움이 되길 바랍니다.
