---
title: "Health Check는 서비스 가용성을 확인해야 한다"
date: "2026-02-11"
tags: [health-check, devops, monitoring]
---

# Health Check는 서비스 가용성을 확인해야 한다

## 문제 발견

Contents Hub 프로젝트에서 `/health` 엔드포인트가 단순히 `{"status": "ok"}`만 반환하고 있었다. 서버 프로세스는 떠있지만 DB 연결이 끊기거나 Redis가 죽어도 health check는 "ok"를 반환했다.

외부 모니터링(UptimeRobot 같은)에서는 200 응답만 보고 "정상"으로 판단하지만, 실제로는:
- DB lock으로 크롤러가 멈춰있거나
- Redis 장애로 캐시가 불가능한 상태

이런 "서버는 살았지만 서비스는 불가"한 상태를 감지하지 못했다.

## 핵심 인사이트

> **Health check는 서버 프로세스가 아니라 "서비스가 제공 가능한지"를 검증해야 한다.**

### 얕은 Health Check의 문제

```python
# Before: 서버만 확인
@app.get("/health")
async def health():
    return {"status": "ok"}
```

이 방식의 문제:
- FastAPI가 HTTP 요청을 받을 수 있는지만 확인
- DB/Redis 연결 여부는 검증 안 함
- 모니터링 시스템이 "정상"이라고 오판하게 만듦

### 개선된 Health Check

```python
# After: 의존성까지 확인
@app.get("/health")
async def health():
    checks = {"server": "ok"}

    # DB 연결 확인
    try:
        async with AsyncSessionLocal() as session:
            await session.execute(text("SELECT 1"))
        checks["db"] = "ok"
    except Exception as e:
        checks["db"] = f"error: {type(e).__name__}"

    # Redis 연결 확인
    try:
        redis_client = aioredis.from_url(settings.redis_url)
        await redis_client.ping()
        checks["redis"] = "ok"
    except Exception as e:
        checks["redis"] = f"error: {type(e).__name__}"

    status = "ok" if all(v == "ok" for v in checks.values()) else "degraded"
    return {"status": status, "checks": checks}
```

## Degraded 상태 패턴

이진 상태(ok/error)만으로는 부분 장애를 표현할 수 없다.

예: Redis는 죽었지만 DB는 살아있으면 읽기 전용 모드로 제한적 서비스 제공이 가능하다.

**3단계 상태 모델**:
- `ok`: 모든 의존성 정상
- `degraded`: 일부 기능 불가, 서비스는 제공 가능
- `error`: 서비스 불가

```json
{
  "status": "degraded",
  "checks": {
    "server": "ok",
    "db": "ok",
    "redis": "error: ConnectionError"
  }
}
```

운영자가 어느 부분이 문제인지 즉시 파악 가능하다.

## 보너스: 의존성 자동 시작

수동 절차("서버 시작 전에 Docker 컨테이너를 먼저 띄워야 한다")는 반드시 누락된다. 특히 드물게 실행하는 절차(서버 재시작)일수록 자동화가 필수다.

```bash
# start_server.sh
check_docker() {
    local db_running=$(docker ps --filter "name=db" --filter "status=running" -q)
    if [ -z "$db_running" ]; then
        echo "Starting docker-compose..."
        docker-compose up -d
        sleep 3
    fi
}

start_server() {
    check_docker  # 의존성 자동 확인/시작
    # ... 서버 시작
}
```

> "사람은 실수하지만 스크립트는 일관적"

## 요약

1. **얕은 health check는 장애를 숨긴다** - "응답한다 ≠ 제대로 작동한다"
2. **의존성을 개별적으로 검증하라** - DB는 `SELECT 1`, Redis는 `PING`
3. **Degraded 상태로 부분 장애를 표현하라** - 불필요한 전체 서비스 중단 방지
4. **의존성 시작을 자동화하라** - 수동 절차는 반드시 누락됨
