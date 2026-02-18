---
title: "Plugin 중복 에러, settings.json이 범인이었다"
date: 2026-02-18
tags: [claude-code, plugin, debugging]
---

# Plugin 중복 에러, settings.json이 범인이었다

## 발단: "Plugin Errors"

`/plugin`을 눌렀을 때 빨간 `×`가 두 개 떠 있었다.

```
1password Plugin · my-plugins · × 1 error
1password Plugin · 1password · × failed to load · 1 error
```

같은 플러그인이 두 번 로드되고 있다. 하나는 `my-plugins` 마켓플레이스에서 설치한 정상 경로, 다른 하나는 `1password`라는 존재하지 않는 마켓플레이스에서 로드를 시도하고 있다.

## 첫 번째 가설: 자동 감지 충돌

Claude Code는 현재 디렉토리에 `.claude-plugin/plugin.json`이 있으면 자동으로 플러그인을 감지한다. 1password 프로젝트 안에서 작업 중이니까, 마켓플레이스 설치본과 자동 감지본이 충돌하는 거라고 생각했다.

그런데 **다른 디렉토리**(`contents_hub`)에서 Claude Code를 열어도 동일한 에러가 발생했다. 자동 감지는 현재 디렉토리에서만 일어나야 하는데?

## 두 번째 가설: 캐시 오염

마켓플레이스에서 설치하면 `~/.claude/plugins/cache/` 아래에 플러그인이 통째로 복사된다. 캐시 안에도 `.claude-plugin/plugin.json`이 있으니, Claude Code가 캐시 디렉토리를 스캔하면서 자동 감지를 한 번 더 하는 것 아닐까?

캐시의 `.claude-plugin/`을 삭제해봤다. 그런데 `disk-cleaner`과 `snapkin`도 캐시에 `.claude-plugin/`이 있었지만 중복 에러가 없었다. **캐시가 원인이 아니었다.**

## 세 번째 가설이자 정답: settings.json의 유령

혹시 설정 파일에 흔적이 남아 있을까? `~/.claude/settings.json`을 grep 해보니:

```json
"1password@1password": true,
"1password@my-plugins": true
```

**있었다.** 이전에 1password 프로젝트 디렉토리에서 작업하면서 자동 감지로 등록된 `1password@1password` 항목이 `settings.json`에 영구 저장되어 있었다. 이후 마켓플레이스에서 정식 설치했지만, 이전 항목은 삭제되지 않았다.

Claude Code는 매 실행마다 이 설정을 읽고, "1password"라는 마켓플레이스에서 플러그인을 찾으려 시도한다. 당연히 존재하지 않는 마켓플레이스이니 `failed to load`. 디렉토리 위치와 무관하게 **전역 설정**이니까 어디서든 에러가 발생한 것이다.

`1password@1password` 한 줄을 삭제하자, 에러가 깨끗이 사라졌다.

## 삽질의 교훈

**증상은 같아도 원인의 계층이 다르다.** 이번 디버깅의 흐름을 정리하면:

| 가설 | 검증 방법 | 결과 |
|------|-----------|------|
| 자동 감지 충돌 | 다른 디렉토리에서 실행 | 여전히 발생 → 기각 |
| 캐시 오염 | 캐시 `.claude-plugin` 삭제 + 다른 플러그인 비교 | 다른 플러그인은 정상 → 기각 |
| settings.json 잔여 항목 | grep으로 설정 파일 검색 | `1password@1password` 발견 → 해결 |

핵심은 **"다른 디렉토리에서도 발생한다"**는 관찰이었다. 이 사실 하나가 로컬 원인(자동 감지, 캐시)을 배제하고, 전역 설정으로 범위를 좁혀줬다.

플러그인 시스템처럼 여러 계층(프로젝트 → 캐시 → 전역 설정)이 겹치는 구조에서는, 문제가 어느 계층에서 발생하는지를 먼저 특정하는 게 가장 빠른 길이다.
