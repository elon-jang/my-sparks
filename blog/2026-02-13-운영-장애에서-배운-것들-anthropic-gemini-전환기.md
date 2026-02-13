---
title: "운영 장애에서 배운 것들 — Anthropic→Gemini 전환기"
date: "2026-02-13"
tags: [devops, deployment, gemini, migration, lessons-learned]
---

# 운영 장애에서 배운 것들 — Anthropic→Gemini 전환기

아침에 다이제스트 메일이 안 왔다. 운영 서버 로그를 열어보니:

```
Client.__init__() got an unexpected keyword argument 'proxies'
```

Anthropic SDK가 httpx 0.28과 호환되지 않아서 다이제스트 생성이 통째로 실패한 것. 여기서 시작된 삽질이 반나절 만에 AI 프로바이더 전환 + 배포 자동화 개선까지 이어졌다.

## 1. 간접 의존성의 숨은 폭탄

httpx 0.28에서 `proxies` 파라미터가 제거됐다. 직접 쓰는 코드에서는 `proxies`를 쓴 적이 없는데? Anthropic SDK 0.18.1이 내부적으로 `httpx.Client(proxies=...)` 를 호출하고 있었다.

```
anthropic 0.18.1 → httpx.Client(proxies=...) → httpx 0.28에서 제거됨
```

**httpx 0.28은 왜 올라갔나?** 며칠 전 FastMCP를 설치하면서 함께 업그레이드됐다. pip는 경고만 출력하고 설치를 완료했기 때문에, 테스트에서도 잡히지 않았다 (테스트에서는 Anthropic API를 mock하니까).

### 교훈

> base library(httpx, requests 등) 버전이 올라가면 **그걸 내부적으로 쓰는 SDK 전체**의 호환성을 확인해야 한다. `pip install`이 성공한다고 런타임에 동작하는 건 아니다.

## 2. 고치지 말고 바꿔라

Anthropic SDK 버전을 올리면 될 일이었지만, 생각해보면:
- 이미 Gemini API 키가 있다 (임베딩에 사용 중)
- Gemini 2.5 Flash는 더 저렴하다 (Anthropic 대비 ~1/8 비용)
- 의존성 그래프가 단순해진다

```python
# Before
from anthropic import Anthropic
client = Anthropic()
response = client.messages.create(model="claude-sonnet-4-20250514", ...)

# After
from google import genai
client = genai.Client(api_key=api_key)
response = client.models.generate_content(model="gemini-2.5-flash", ...)
```

기존 API 키를 재사용하고, 코드 변경은 `_generate_summary` 함수 하나. 테스트도 mock 대상만 바꾸면 됐다. 30분 작업.

### 교훈

> 버그를 고치는 것보다 **문제 자체를 없애는 게** 나을 때가 있다. SDK A + SDK B가 같은 base library의 다른 버전을 요구하면, pip로는 해결 불가 — 둘 중 하나를 교체해야 한다.

## 3. 같은 실수를 두 번 하면 자동화해야 한다

Gemini로 전환한 코드를 배포했더니, 이번엔 다른 에러:

```
column subscriptions.embed_contents does not exist
```

며칠 전에 추가한 `embed_contents` 컬럼의 DB 마이그레이션을 운영에 적용하지 않았다. **이 실수는 두 번째였다.** 2주 전에도 `embedding` 컬럼 마이그레이션을 누락해서 같은 장애가 발생했고, 그때 CLAUDE.md에 체크리스트를 추가했다.

체크리스트가 있어도 안 본다. "대부분의 배포는 마이그레이션 불필요"라는 문구가 "이번에도 안 해도 되겠지" 심리를 만든다.

### 자동화한 것

deploy.sh에 마이그레이션 자동 감지를 추가했다:

```bash
# 로컬 alembic HEAD vs 운영 DB revision 비교
LOCAL_HEAD=$(ls alembic/versions/[0-9]*.py | sort -V | tail -1 | ...)
REMOTE_REV=$(ssh ... "docker exec postgres psql -t -c 'SELECT version_num FROM alembic_version'")

if [ "$LOCAL_NUM" -gt "$REMOTE_NUM" ]; then
    # 새 이미지로 임시 컨테이너 실행 → alembic upgrade head
    ssh ... "docker run --rm --env-file .env --network pgvector_default $IMAGE alembic upgrade head"
fi
```

핵심은 **순서**다:
1. `docker pull` — 새 이미지 다운로드
2. `alembic upgrade head` — DB 스키마 업데이트 (임시 컨테이너)
3. `docker-compose up -d` — 새 코드로 서비스 재시작

재시작 후에 마이그레이션하면 안 된다. 새 코드가 없는 컬럼을 참조하면서 즉시 크래시.

### 교훈

> 같은 실수가 반복되면 **문서화가 아니라 자동화**가 정답이다. 체크리스트는 실패하는 것이 기본값.

## 4. AI 모델은 코드보다 빠르게 죽는다

Gemini 2.0 Flash로 전환했는데, 공식 문서를 열어보니:

> `gemini-2.0-flash` is deprecated and will be shut down on **March 31, 2026**

방금 배포한 코드를 즉시 수정. `gemini-2.5-flash`(stable)로 업그레이드.

### 교훈

> AI 모델은 소프트웨어 패키지와 다르다. LTS 같은 건 없다. 배포 전 **deprecation timeline 확인** 필수. 이상적으로는 모델명을 환경변수로 외부화해서 코드 수정 없이 전환 가능하게.

## 5. 수동으로 DB를 고치면 반드시 흔적을 남겨라

`embed_contents` 컬럼을 `ALTER TABLE`로 수동 추가한 뒤, Alembic의 `alembic_version` 테이블을 업데이트하지 않았다. revision이 `003`에 머물러 있어서, 다음 배포 시 deploy.sh가 "마이그레이션 필요"로 오판할 뻔했다.

```sql
-- 수동 DDL 후 반드시 함께 실행
UPDATE alembic_version SET version_num = '005';
```

### 교훈

> 수동 스키마 변경 후 **alembic_version 동기화는 즉시 함께**. 아니면 다음 자동 마이그레이션이 이미 존재하는 컬럼을 다시 추가하려다 실패한다.

## 타임라인

| 시간 | 이벤트 |
|------|--------|
| 09:00 | 아침 다이제스트 미수신 확인 |
| 09:10 | 운영 로그에서 `proxies` 에러 발견 |
| 09:30 | Gemini API로 전환 결정 |
| 10:00 | 코드 변경 + 테스트 수정 완료 |
| 10:15 | v1.0.5 배포 → `embed_contents` 컬럼 누락으로 2차 장애 |
| 10:20 | 수동 ALTER TABLE + alembic_version 동기화 |
| 10:25 | 서비스 복구, 다이제스트 발송 확인 |
| 10:30 | gemini-2.0-flash deprecation 발견 → 2.5-flash 업그레이드 |
| 10:35 | v1.0.6 배포 |
| 11:00 | deploy.sh 마이그레이션 자동화 구현 |

반나절의 삽질이었지만, 결과적으로:
- AI 프로바이더가 더 저렴한 것으로 전환됨
- 배포 파이프라인에 마이그레이션 자동화가 추가됨
- 5개의 레슨이 문서화됨

**장애는 시스템을 개선하는 가장 확실한 계기다.**
