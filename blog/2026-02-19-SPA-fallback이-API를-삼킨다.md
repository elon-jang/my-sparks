---
title: "SPA fallback이 API를 삼킨다"
date: 2026-02-19
tags: [fastapi, spa, frontend, debugging]
---

# SPA fallback이 API를 삼킨다

FastAPI + React SPA를 한 서버에서 서빙할 때 마주친 함정. Excel 템플릿 다운로드가 계속 깨지는 증상의 원인을 추적한 기록.

## 증상

프론트엔드에서 "입력 양식 다운로드" 버튼을 눌렀더니 `.xlsx` 파일이 다운로드되긴 하는데, Excel에서 열면 "file format or file extension is not valid" 에러가 뜬다. 파일 크기는 398바이트. 열어보면 `<!doctype html>` — index.html이었다.

## 디버깅 과정

**1차 시도: `<a href download>` → 빈 페이지**

SPA의 클라이언트 라우팅이 `/api/templates/income` 경로를 가로챈다. React Router가 매칭되는 라우트가 없어서 빈 페이지를 렌더링.

**2차 시도: `window.open()` → 여전히 HTML**

Vite dev server의 SPA history fallback이 `Accept: text/html` 헤더를 보고 index.html을 반환. `window.open()`은 브라우저가 네비게이션으로 처리하기 때문에 HTML을 기대한다.

**3차 시도: `fetch()` + blob → 파일 다운로드됨, 하지만 깨짐**

```typescript
const res = await fetch(path);
const blob = await res.blob();
const url = URL.createObjectURL(blob);
// ... programmatic download
```

이제 Vite 프록시를 제대로 탄다. 그런데 여전히 HTML이 내려온다.

## 근본 원인

FastAPI의 SPA fallback 코드:

```python
@app.get("/{full_path:path}")
async def serve_spa(full_path: str):
    file_path = _frontend_dist / full_path
    if file_path.is_file():
        return FileResponse(str(file_path))
    return FileResponse(str(_frontend_dist / "index.html"))
```

`frontend/dist/` 디렉토리가 존재하면 이 catch-all 라우트가 등록된다. 문제는 **등록되지 않은 모든 경로**를 잡는다는 것. `/api/templates/income`이 아직 등록되지 않은 상태(서버 미재시작)에서 이 라우트가 `index.html`을 반환했다.

개발 환경에서는 Vite 프록시가 먼저 처리해서 문제가 안 보였고, `frontend/dist`가 있는 프로덕션 빌드 환경에서만 발생하는 조건부 버그였다.

## 수정

```python
@app.get("/{full_path:path}")
async def serve_spa(full_path: str):
    if full_path.startswith(("api/", "auth/")):
        return JSONResponse(status_code=404, content={"detail": "Not found"})
    file_path = _frontend_dist / full_path
    if file_path.is_file():
        return FileResponse(str(file_path))
    return FileResponse(str(_frontend_dist / "index.html"))
```

추가로 브라우저 캐시 문제도 있었다. 이전에 HTML을 받았던 응답이 캐싱되어 서버 수정 후에도 동일한 HTML을 반환:

```typescript
const res = await fetch(path, { cache: "no-store" });
```

## 교훈

SPA와 API를 같은 서버에서 서빙할 때:

1. **SPA catch-all에는 반드시 API 경로 제외 조건을 넣어라.** `/api/`, `/auth/` 등 백엔드 네임스페이스는 명시적으로 빼야 한다.
2. **curl로 되는데 브라우저에서 안 되면** — Accept 헤더, 캐시, SPA fallback 순서로 의심하라.
3. **파일 다운로드는 `fetch` + blob이 가장 안전하다.** `<a href>`, `window.open()`은 SPA 환경에서 예측 불가능하다.
4. **`frontend/dist` 존재 여부로 동작이 달라지는 조건부 코드**는 개발/프로덕션 환경 차이를 만든다. 두 환경 모두에서 테스트해야 한다.
