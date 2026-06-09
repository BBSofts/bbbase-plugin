# 주기적 리셋 + 감사로그

```
BASE_URL : https://api.bbbase.io
```
둘 다 **JWT 필요** (`Authorization: Bearer {accessToken}`).

> 정확한 경로·필드·타입은 `{BASE_URL}/docs-json`(라이브 OpenAPI)이 권위다. 아래는 그 위에
> **동작·의미**를 더한 것 — 어긋나면 `/docs-json` 을 믿어라. (404 면 Swagger 비활성.)

## 리셋 잡 — 특정 컬럼을 매일/매주/매달 기본값으로 초기화

"일일 베스트타임", "오늘 시도 횟수" 처럼 주기적으로 리셋되는 값에 쓴다. 서버가 UTC
00:00 에 BullMQ 로 일괄 처리하며, 서버 재시작에도 등록된 잡이 자동 복구된다.

### 등록 — `POST /projects/{projectId}/reset-jobs`

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| schemaId | string | ✅ | 초기화할 컬럼의 schema id |
| frequency | `DAILY`\|`WEEKLY`\|`MONTHLY` | ✅ | 주기 |
| resetTo | any | ✅ | 초기화 값 |

```bash
# daily_best_time 을 매일 자정 999999 로 리셋
curl -X POST https://api.bbbase.io/projects/{projectId}/reset-jobs \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "schemaId": "{schemaId}", "frequency": "DAILY", "resetTo": 999999 }'
```

> schemaId 는 `GET /projects/{projectId}/schemas` 에서 해당 컬럼의 `id` 를 찾아 쓴다.
> 모든 리셋은 UTC 00:00 실행(타임존 지원은 보류). 대량 리셋은 per-record 감사로그를
> 남기지 않는다(테이블 폭증 방지) — `ResetJob.lastRunAt` 이 실행 이력.

### 목록 / 수정 / 삭제

```bash
GET    /projects/{projectId}/reset-jobs
GET    /projects/{projectId}/reset-jobs/{jobId}
PATCH  /projects/{projectId}/reset-jobs/{jobId}
DELETE /projects/{projectId}/reset-jobs/{jobId}
```

### 전형적 흐름 — "일일 베스트타임" 만들기
1. `POST /schemas` 로 `daily_best_time` 생성(`type: NUMBER, compareMode: MIN, defaultValue: 999999`) → schemaId
2. `POST /reset-jobs` 로 그 schemaId, `frequency: DAILY, resetTo: 999999` 등록
3. 게임은 기록 저장 시 `best_time`(전체)과 `daily_best_time`(일일)을 함께 PUT — 일일 값만 매일 리셋됨

## 감사로그 — 레코드 변경 이력 조회

`GET /projects/{projectId}/audit-logs` (최신순). 레코드가 삭제돼도 로그는 남는다
(audit_logs 는 레코드로의 FK 가 없음 — 설계상 로그가 레코드보다 오래 산다).

| 쿼리 | 설명 |
|---|---|
| userId | 특정 유저 필터 |
| recordId | 특정 레코드 필터 |
| action | `CREATE`\|`UPDATE`\|`DELETE` 필터 |
| limit | 기본 50, 최대 200 |
| cursor | 페이지네이션 커서 |

```bash
curl "https://api.bbbase.io/projects/{projectId}/audit-logs?userId={userId}&limit=10" \
  -H "Authorization: Bearer {accessToken}"
```

> 주의: 대량 리셋 잡은 감사로그를 남기지 않으므로, 리셋으로 인한 값 변화는 여기에
> 안 나타난다. 게임 클라이언트의 개별 `PUT .../record` 만 기록된다.

## 헬스체크 (인증 불필요)
```bash
curl https://api.bbbase.io/health        # db+redis 포함, 장애 시 503
curl https://api.bbbase.io/health/live   # 프로세스 생존만(항상 200) — liveness probe 용
```
