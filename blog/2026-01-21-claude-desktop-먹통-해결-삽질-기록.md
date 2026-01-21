---
title: "Claude Desktop이 갑자기 먹통이 됐을 때 - 삽질 3시간의 기록"
date: "2026-01-21"
tags: [claude-desktop, troubleshooting, mcp, macos]
---

## Claude Desktop이 갑자기 먹통이 됐을 때 - 삽질 3시간의 기록

### 어느 날 갑자기...

평소처럼 Claude Desktop을 켜고 "영수증 대상자 목록 보여줘"라고 입력했습니다.

그런데 이게 웬일? 메시지를 보내자마자 화면이 휙 하고 초기화면으로 돌아가버리는 겁니다.

"어? 내가 뭘 잘못 쳤나?"

다시 시도. "오늘 날씨는?"

또 휙. 초기화.

아무 말이나 쳐도 대화창이 리셋되는 상황. 이게 뭐지?

---

### 범인을 찾아서

#### 용의자 1: MCP 서버

저는 기부금 영수증을 자동 발행하는 MCP 서버를 Docker로 돌리고 있었거든요. "혹시 이놈이 문제인가?"

```bash
# MCP 서버 직접 테스트
echo '{"jsonrpc":"2.0",...}' | docker run -i oikos-receipt:latest
```

결과: 정상 작동. 94명 대상자, 5천만원... 데이터 잘 나옴.

범인 아님.

#### 용의자 2: 로그 파일

진짜 범인을 찾으려면 로그를 봐야죠.

```bash
tail ~/Library/Logs/Claude/claude.ai-web.log
```

```
[COMPLETION] Request failed Error: Internal server error
[REACT_QUERY_CLIENT] QueryClient error: Error: Invalid authorization
```

아하! Internal server error와 Invalid authorization이 반복되고 있었습니다.

그런데 이상한 점: 웹(claude.ai)은 잘 됨. 똑같은 계정인데?

이건 서버 문제가 아니라 **앱 내부 데이터가 꼬인 것**!

---

### 삽질의 시작

**시도 1: 캐시 삭제**

```bash
rm -rf ~/Library/Application\ Support/Claude/Cache
```
→ 안 됨

**시도 2: 앱 재시작**

```bash
pkill -f "Claude" && open -a "Claude"
```
→ 안 됨

**시도 3: 로그아웃 후 재로그인**

Settings → Log out → 다시 로그인
→ 안 됨

**시도 4: 일부 폴더 삭제**

Session Storage, Local Storage 등 삭제
→ 그래도 안 됨

점점 멘탈이...

---

### 결국 답은 '밀어버리기'

미련 없이 전체 초기화를 결심했습니다.

단, MCP 서버 설정(`claude_desktop_config.json`)은 살려야 하니까 백업 먼저!

```bash
# 1. 앱 종료
pkill -f "Claude"

# 2. 설정 파일만 백업
cp ~/Library/Application\ Support/Claude/claude_desktop_config.json /tmp/

# 3. 싹 다 날리기
rm -rf ~/Library/Application\ Support/Claude/*

# 4. 설정 파일 복원
cp /tmp/claude_desktop_config.json ~/Library/Application\ Support/Claude/

# 5. 앱 실행
open -a "Claude"
```

로그인하고... 메시지 전송...

**됐다!!!**

---

### 뭐가 문제였을까?

`~/Library/Application Support/Claude/` 폴더 안에는 이런 것들이 있습니다:

```
├── Session Storage/     ← 세션 데이터
├── Local Storage/       ← 로컬 저장소
├── IndexedDB/           ← 데이터베이스
├── Cookies              ← 쿠키
├── blob_storage/        ← 바이너리 데이터
└── claude_desktop_config.json  ← MCP 설정 (이것만 살리면 됨!)
```

이 중 세션이나 인증 관련 파일이 손상되면서 "로그인은 됐는데 API 호출이 안 되는" 기묘한 상태가 된 거죠.

마치 집 열쇠는 있는데 문이 안 열리는 느낌?

---

### 정리: 이럴 때 이렇게!

| 증상 | 해결책 |
|------|--------|
| 웹은 되는데 앱만 안 됨 | 앱 데이터 문제 → 전체 초기화 |
| 캐시 삭제해도 안 됨 | Session/Local Storage도 삭제 필요 |
| 로그아웃해도 안 됨 | 인증 데이터 자체가 손상됨 |

### 최종 해결 명령어 (복붙용)

```bash
pkill -f "Claude"
cp ~/Library/Application\ Support/Claude/claude_desktop_config.json /tmp/
rm -rf ~/Library/Application\ Support/Claude/*
cp /tmp/claude_desktop_config.json ~/Library/Application\ Support/Claude/
open -a "Claude"
```

---

### 교훈 세 줄 요약

1. **부분 삭제는 미련입니다.** 안 되면 과감히 밀어버리세요.
2. **설정 파일만 백업하면** MCP 서버 설정은 살릴 수 있어요.
3. **로그 파일은 친구입니다.** `~/Library/Logs/Claude/`를 기억하세요.

다음에 또 이런 일이 생기면 30초 만에 해결할 수 있겠죠?

---

이 글이 도움이 됐다면, 당신의 Claude Desktop도 무사하길 바랍니다!
