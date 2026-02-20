---
title: "백엔드 없이 실시간 동기화 만들기 — Cloudflare Workers + KV로 PWA 멀티디바이스 sync 구현"
date: "2026-02-20"
tags: [cloudflare-workers, pwa, sync, javascript]
---

# 백엔드 없이 실시간 동기화 만들기

## 발단: "왜 폰이랑 PC가 달라요?"

학습 앱을 만들고 배포했다. 단일 HTML 파일, 외부 의존성 제로, Cloudflare Pages에 올려두면 그걸로 끝. 완벽했다.

그런데 사용자가 물었다.

> "여러 디바이스에서 학습을 한건데, 학습 진행 사항을 동기화할 수 없어요?"

아. localStorage는 브라우저마다 완전히 분리된다. PC에서 챕터 3까지 학습했는데 모바일을 열면 처음부터다. 이건 앱이 아니라 기억상실증 환자다.

---

## 선택지를 줄여나가는 과정

동기화 구현 방법은 여러 가지다.

- **Firebase / Supabase**: 계정 기반, 제대로 된 auth. 근데 외부 서비스 의존성이 생기고, 지금 앱 철학(zero deps)과 맞지 않는다.
- **Export/Import**: 파일 다운받아서 다른 기기에서 업로드. 구현은 쉬운데... 솔직히 아무도 안 한다.
- **Cloudflare Workers + KV**: 이미 Cloudflare Pages 쓰고 있으니 같은 플랫폼. Workers는 엣지 함수, KV는 글로벌 분산 키-값 저장소.

세 번째가 답이었다. 같은 생태계, 무료 한도 충분, 구현 단순.

**식별자는 이메일**로 결정했다. 별도 회원가입 없이 "이메일 = 내 계정"이라는 직관적인 UX. 보안? 소수 사용자의 학습 앱에선 충분한 수준이다.

---

## 구현: 생각보다 단순했다

Worker 코드는 50줄이면 충분하다.

```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const match = url.pathname.match(/^\/sync\/([a-f0-9]{64})$/);
    if (!match) return json({ error: 'Not found' }, 404);

    const key = match[1]; // SHA-256(email)

    if (request.method === 'GET') {
      const raw = await env.ECODOM_SYNC.get(key);
      return json({ data: raw ? JSON.parse(raw) : null });
    }
    if (request.method === 'POST') {
      const body = await request.json();
      await env.ECODOM_SYNC.put(key, JSON.stringify(body));
      return json({ ok: true });
    }
  }
};
```

클라이언트에선 이메일을 SHA-256으로 해시해서 키로 쓴다. 이메일 평문을 서버에 보내지 않아도 되고, URL에 노출되지 않는다.

```javascript
async function hashEmail(email) {
  const buf = await crypto.subtle.digest(
    'SHA-256',
    new TextEncoder().encode(email.trim().toLowerCase())
  );
  return Array.from(new Uint8Array(buf))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
}
```

`crypto.subtle`은 브라우저 내장 API라 외부 라이브러리 불필요.

---

## 첫 번째 함정: Service Worker 캐시

배포 완료. 브라우저에서 열었더니... 아무것도 안 바뀌었다.

당연하다. PWA는 Service Worker가 모든 리소스를 캐시한다. `Cache-Control: no-store`를 눌러도 SW 캐시는 별개다. 개발자도구의 "Disable cache" 체크박스도 SW 캐시엔 적용 안 된다.

**해결책은 단순하다: 캐시 버전 번호를 올린다.**

```javascript
const CACHE_NAME = 'ecodom-v4'; // v3 → v4
```

새 버전의 SW가 설치되면 이전 캐시를 삭제한다. 사용자는 강제 새로고침(Cmd+Shift+R) 또는 Application → Service Workers → Unregister 후 새로고침.

> **교훈**: PWA 업데이트는 배포 후 캐시 버전 변경이 필수다. CI/CD에 자동화하면 더 좋다.

---

## 두 번째 함정: 탭 닫는 순간 fetch가 죽는다

자동 동기화를 구현했다. 로직은 완벽해 보였다.

- 앱 열기 → `autoSyncPull()`
- 탭 닫기/이탈 → `autoSyncPush()`

```javascript
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden') autoSyncPush();
});
```

그런데 PC는 50%, 모바일은 38%가 표시됐다. 데이터가 안 올라간 것이다.

원인은 명확하다. `visibilitychange` → `hidden`이 발생하는 순간 브라우저는 페이지를 정리하기 시작한다. 비동기 `fetch`가 완료되기 전에 프로세스가 종료된다.

**해결책 1: `keepalive: true`**

```javascript
const res = await fetch(SYNC_WORKER_URL + '/sync/' + hash, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(payload),
  keepalive: true, // 페이지 종료 후에도 요청 완료 보장
});
```

`keepalive: true`는 페이지가 unload된 후에도 브라우저가 요청을 완료하도록 보장한다. 단, body 크기 제한이 64KB이므로 큰 데이터는 주의.

**해결책 2: `pagehide` 이벤트 추가**

```javascript
window.addEventListener('pagehide', autoSyncPushImmediate);
```

`visibilitychange`는 탭 전환 시 발생하지만, iOS Safari에서 앱을 종료하거나 다른 앱으로 넘어갈 때는 `pagehide`가 더 신뢰할 수 있다. 두 이벤트를 모두 listen해야 모바일 환경을 제대로 커버한다.

> **교훈**: PWA에서 "앱 닫기" 시점의 동작은 `visibilitychange`만으론 부족하다. `pagehide` + `keepalive` 조합이 필수다.

---

## 세 번째 발견: 닫을 때 말고, 변경될 때 push하라

그래도 여전히 갭이 있었다. PC에서 학습 중인데, 탭을 안 닫으면 push가 안 된다.

발상을 바꿨다. **탭을 닫을 때가 아니라 데이터가 변경될 때마다 push하면 된다.**

```javascript
function saveProgress() {
  localStorage.setItem(STORAGE_KEYS.progress, JSON.stringify(state.progress));
  autoSyncPush(); // 여기!
}
function saveQuiz() {
  localStorage.setItem(STORAGE_KEYS.quiz, JSON.stringify(state.quizScores));
  autoSyncPush();
}
```

`autoSyncPush`는 800ms 디바운스가 걸려 있어서 연속 저장 시 한 번만 요청한다.

이제 흐름이 깔끔해졌다:

```
섹션 완료 → saveProgress() → 800ms 후 push
퀴즈 완료 → saveQuiz() → 800ms 후 push
다른 기기에서 앱 열기 → autoSyncPull() → 최신 데이터 반영
```

---

## 충돌 처리: merge 전략

두 기기에서 동시에 학습하면 어떻게 되나? 더 많이 학습한 쪽이 이겨야 한다.

```javascript
function mergeData(local, remote) {
  // progress: 완료된 섹션은 유지 (OR 병합)
  const progress = Object.assign({}, remote.progress || {});
  for (const ch in local.progress) {
    for (const sec in local.progress[ch]) {
      if (local.progress[ch][sec]) progress[ch][sec] = true;
    }
  }
  // quizScores: 높은 점수 유지
  // leitner: 높은 박스 번호 유지 (더 많이 복습한 쪽)
  // streak: 높은 카운트 유지
  return { progress, quizScores, leitner, streak };
}
```

학습 앱에서 "퇴보"는 없다. 이미 완료한 섹션이 미완료로 돌아가거나, 높은 퀴즈 점수가 낮아지면 안 된다.

---

## 최종 아키텍처

```
[디바이스 A]                    [Cloudflare KV]              [디바이스 B]
localStorage ──saveProgress()──▶ push(keepalive) ──────────▶ pull(앱 시작)
                                 key: SHA256(email)
             ◀─────────────────  merge(local, remote) ◀──────
```

Worker 1개, KV 네임스페이스 1개, 추가 서버 없음. 배포는 `wrangler deploy` 한 줄.

---

## 핵심 요약

| 상황 | 해결책 |
|------|--------|
| SW 캐시로 업데이트 안 됨 | `CACHE_NAME` 버전 올리기 |
| 탭 닫을 때 fetch 실패 | `keepalive: true` + `pagehide` 이벤트 |
| 모바일 iOS sync 누락 | `pagehide` 추가 (visibilitychange 불충분) |
| 실시간 동기화 지연 | 데이터 변경 즉시 push (저장 함수에 훅) |
| 충돌 처리 | merge 전략 (OR/max 기반) |

백엔드 없이도 실용적인 실시간 동기화는 충분히 가능하다. Cloudflare Workers + KV의 무료 한도는 개인/소규모 앱에선 걱정할 필요가 없다.

정적 사이트라서 뭔가 못 한다는 건 점점 옛말이 되고 있다.
