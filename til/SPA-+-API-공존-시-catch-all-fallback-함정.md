---
id: "1739926200-spa-fallback"
title: "SPA + API 공존 시 catch-all fallback 함정"
category: "til"
tags: [fastapi, spa, debugging, architecture]
created: "2026-02-19T08:30:00+09:00"
source: "Oikos Finance 프로젝트 디버깅"
blog_link: "blog/2026-02-19-SPA-fallback이-API를-삼킨다.md"
confidence: 4
connections: []
review_count: 0
last_reviewed: null
---

# SPA + API 공존 시 catch-all fallback 함정

> **Blog**: [SPA fallback이 API를 삼킨다](../blog/2026-02-19-SPA-fallback이-API를-삼킨다.md)

## 핵심

SPA와 API를 같은 서버에서 서빙할 때, `/{full_path:path}` 같은 catch-all 라우트가 등록되지 않은 `/api/*` 경로까지 가로채서 `index.html`을 반환하는 문제.

## Key Points

1. **SPA catch-all에 `/api/` 제외는 필수** — 빠뜨리면 미등록 API 경로가 HTML을 반환
2. **파일 다운로드는 `fetch` + blob이 가장 안전** — `<a href>`, `window.open()`은 SPA 환경에서 예측 불가
3. **`frontend/dist` 존재 여부로 동작이 달라지는 조건부 코드 주의** — 개발/프로덕션 환경 차이 발생

## 해결 패턴

```python
@app.get("/{full_path:path}")
async def serve_spa(full_path: str):
    if full_path.startswith(("api/", "auth/")):
        return JSONResponse(status_code=404, content={"detail": "Not found"})
    # ... SPA fallback
```

```typescript
// 브라우저 파일 다운로드
const res = await fetch(path, { cache: "no-store" });
const blob = await res.blob();
const url = URL.createObjectURL(blob);
// programmatic <a> click + setTimeout revokeObjectURL
```

## Q&A

**Q1: curl로는 정상인데 브라우저에서만 깨지는 이유는?**
A: Accept 헤더 차이, 브라우저 캐시, SPA fallback 세 가지가 원인. curl은 프록시를 직접 타지만 브라우저는 SPA 라우터/캐시를 거친다.

**Q2: 개발 환경(Vite)에서는 왜 문제가 안 보이나?**
A: Vite dev server의 프록시가 `/api` 요청을 먼저 처리하기 때문. `frontend/dist`가 없으면 SPA fallback 자체가 등록되지 않는다.

**Q3: `cache: "no-store"`가 필요한 이유는?**
A: 이전에 HTML을 받았던 응답이 브라우저에 캐싱되어 서버 수정 후에도 동일한 HTML을 반환하기 때문.
