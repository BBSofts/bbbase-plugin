# 리그 (League) — 티어 사다리 + 주기 승격/강등

```
BASE_URL : https://api.bbbase.io
```

> 정확한 경로·필드·타입은 `{BASE_URL}/docs-json`(라이브 OpenAPI)이 권위다. 아래는 그 위에
> **동작·의미·함정**을 더한 것. (`/docs-json` 이 404 면 이 문서를 사용.)

## 개념

브론즈/실버/골드 같은 **티어**가 올라가고, 주기마다 같은 리그 안에서 **승격/강등**되며, 티어(또는 방) 안에서 **점수 랭킹**이 매겨진다. 대부분 기존 인프라 재사용이다.

- **티어** = `league_tier`(NUMBER) 컬럼. 0부터. 이름(브론즈…)은 **클라이언트에서 매핑**(서버는 숫자만 안다).
- **점수** = `league_points`(NUMBER) 컬럼. 리그 전용 점수 API 가 **없다** — 평소처럼 `PUT .../record` 로 저장하면 자동 반영(리더보드와 동일).
- **랭킹판** = 리그 등록 시 **자동 생성되는 리더보드**(groupKey = 티어 또는 방). 별도 등록 불필요.
- 주기(일/주/월)마다 서버가 그룹 내 순위로 상위 승격(`tier+1`)/하위 강등(`tier-1`) 후 점수를 리셋한다.

### 두 모델
- **티어 풀**(기본) — 한 티어 = 하나의 큰 랭킹. `cohortCol` 생략.
- **코호트**(Duolingo 식) — 한 티어를 `cohortSize` 명씩 방으로 쪼개 방 안에서 승강. `cohortCol` 지정(예 `"league_cohort"`, 없으면 STRING 자동 등록).

## 사전 준비 — 스키마 (운영자 JWT)

대상 scope(기본 `user`)에 NUMBER 컬럼 두 개가 필요하다. `compareMode` 는 게임 룰에 맞게(`MAX`=이번 주기 최고점, `INCREMENT`=누적).

```bash
curl -X POST https://api.bbbase.io/projects/{projectId}/schemas \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "columnName": "league_points", "type": "NUMBER", "defaultValue": 0, "compareMode": "MAX" }'
curl -X POST https://api.bbbase.io/projects/{projectId}/schemas \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "columnName": "league_tier", "type": "NUMBER", "defaultValue": 0 }'
```
> 코호트 방 컬럼(`league_cohort`)은 직접 등록하지 않는다 — 리그 등록 시 자동 생성·관리된다.

## 1. 정의 관리 (JWT 필요)

> ⚠️ **등록은 운영자(JWT) 작업이다 — 셋업을 누가 할지 개발자에게 먼저 물어라.**
> ① 내가(에이전트) CLI 로 직접(→ 운영자 로그인 정보 필요), ② 개발자가 대시보드에서 직접. (SKILL.md "4.5 운영자 셋업" 참고.)

### 등록 — `POST /projects/{projectId}/leagues`

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| name | string | ✅ | 표시 이름 (프로젝트 내 유일) |
| tierCount | int(≥2) | ✅ | 티어 칸 수 |
| frequency | `DAILY`\|`WEEKLY`\|`MONTHLY` | ✅ | 승강 주기(UTC 00:00) |
| promoteCount / demoteCount | int | ⛓ | 그룹 내 상위/하위 N명 |
| promotePct / demotePct | number(0~100) | ⛓ | 그룹 내 상위/하위 X% |
| entityType | string | ❌ | 기본 `user` |
| pointsCol / tierCol | string | ❌ | 기본 `league_points` / `league_tier` (둘 다 NUMBER) |
| cohortCol | string | ❌ | 지정 시 코호트 모델(자동 STRING 등록) |
| cohortSize | int(≥2) | ❌ | 방 정원. 기본 30 |
| resetPointsTo | any | ❌ | 승강 후 점수 초기화 값. 기본 0 |

> ⛓ 승강 규칙(promote/demote의 count 또는 pct)은 **최소 1개 필수**. 같은 방향에 count·pct 둘 다 주면 **count 우선**.

```bash
# 티어 풀: 5티어, 매주, 상위 20% 승격 / 하위 20% 강등
curl -X POST https://api.bbbase.io/projects/{projectId}/leagues \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "name": "메인 리그", "tierCount": 5, "frequency": "WEEKLY", "promotePct": 20, "demotePct": 20 }'

# 코호트: 30명 방, 방 안 상위 7명 승격 / 하위 7명 강등
curl -X POST https://api.bbbase.io/projects/{projectId}/leagues \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "name": "코호트 리그", "tierCount": 5, "frequency": "WEEKLY", "cohortCol": "league_cohort", "cohortSize": 30, "promoteCount": 7, "demoteCount": 7 }'
```
응답 `data.id` 가 **leagueId**, `data.leaderboardId` 가 자동 생성된 랭킹판. 등록 시 기존 레코드 점수를 자동 백필한다.

> **한 점수 컬럼은 한 리그 전용**(롤의 일반/자유 랭크처럼 리그마다 컬럼이 달라야 함). 같은 pointsCol 로 또 등록하면 `409 LEAGUE_DUPLICATE`.

### 목록/단건/수정/삭제 + 수동 트리거
```bash
GET    /projects/{projectId}/leagues
GET    /projects/{projectId}/leagues/{id}
PATCH  /projects/{projectId}/leagues/{id}        # name/frequency/승강규칙/resetPointsTo 만
DELETE /projects/{projectId}/leagues/{id}        # 자동 생성 리더보드도 함께 정리
POST   /projects/{projectId}/leagues/{id}/run    # 수동 승강 1사이클 (JWT) → { promoted, demoted }
```
> 점수/티어/코호트 컬럼·entityType·tierCount 는 생성 후 불변(랭킹 구조 보호). 바꾸려면 새 리그 등록.
> 에러: pointsCol/tierCol 미존재·NUMBER 아님/cohortCol STRING 아님/승강 규칙 0개 → `INVALID_LEAGUE_CONFIG`.

## 2. 게임 클라이언트 조회 (API 키 필요)

`X-API-Key: {API_KEY}` 헤더.

```bash
# 내 현황 — tier, (코호트면 cohort), 그룹 내 rank/score/total/percentile
curl "https://api.bbbase.io/projects/{projectId}/leagues/{leagueId}/me/{userId}" -H "X-API-Key: {API_KEY}"

# 내 그룹 랭킹 — 내 티어(또는 방) 내 top-N (리더보드 ranks 형태)
curl "https://api.bbbase.io/projects/{projectId}/leagues/{leagueId}/ranks/{userId}?limit=30" -H "X-API-Key: {API_KEY}"

# 승급 연출 본 뒤 결과 확인 처리(seen=true) — 다음 조회부터 안 뜸
curl -X POST "https://api.bbbase.io/projects/{projectId}/leagues/{leagueId}/me/{userId}/ack" -H "X-API-Key: {API_KEY}"
```
```json
{ "success": true, "data": {
  "leagueId": "...", "entityId": "u_123", "tier": 2, "cohort": "t2-p5-r0",
  "rank": 4, "score": 1500, "total": 30, "percentile": 13.3,
  "lastResult": { "period": 5, "tierFrom": 1, "tierTo": 2, "change": "promote",
                  "rank": 3, "groupSize": 30, "prevRank": 8, "seen": false } } }
```
> **승급 연출(듀오링고식):** `lastResult` 는 지난 사이클 결과. `change=="promote" && !seen` 이면 접속 시 승급 애니메이션을 띄우고, 본 뒤 `.../me/{userId}/ack` 로 확인 처리한다. `change`=promote/demote/stay, `prevRank` 로 "8등→3등" 순위 변화도 연출 가능. 결과 컬럼(`league_last_result`)도 서버 관리라 직접 쓰지 말 것.

## 3. 점수 갱신은 자동
```bash
curl -X PUT https://api.bbbase.io/projects/{projectId}/entities/user/{userId}/record \
  -H "Content-Type: application/json" -H "X-API-Key: {API_KEY}" -H "Authorization: Bearer {gameUserToken}" \
  -d '{ "data": { "league_points": 250 } }'
```
- `compareMode` 그대로 적용. **`league_tier`/`league_cohort` 는 서버가 관리** — 게임 클라가 직접 쓰지 말 것(승강 잡이 덮어쓴다).

## 전형적 흐름
1. (1회, JWT) `league_points`·`league_tier` 스키마 → `POST /leagues` 등록 → leagueId 저장
2. (런타임, API키) 게임은 평소처럼 `PUT .../record` 로 `league_points` 저장 — 랭킹 자동 반영
3. (런타임, API키) `GET .../leagues/{id}/me/{userId}` 로 내 티어/순위, `.../ranks/{userId}` 로 그룹 랭킹 표시
4. 승강은 서버가 주기마다 자동 — 티어 이름(브론즈…)은 `league_tier` 숫자로 클라에서 매핑
