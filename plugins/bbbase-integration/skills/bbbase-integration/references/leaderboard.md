# 리더보드 / 랭킹 (등록형)

```
BASE_URL : https://api.bbbase.io
```

> 정확한 경로·필드·타입은 `{BASE_URL}/docs-json`(라이브 OpenAPI)이 권위다. 아래 표는
> 그 위에 **동작·의미·함정**을 더한 것 — 표가 라이브와 어긋나면 `/docs-json` 을 믿어라.
> (`/docs-json` 이 404 면 Swagger 비활성이니 이 문서를 사용한다.)

## 개념 — 사전 등록형

랭킹을 **미리 등록**해두면 대규모 유저에서도 전용 score 테이블 + 인덱스로 정렬·순위가
빠르게 처리된다. 즉석으로 임의 컬럼을 정렬하는 옛 방식은 제거됐다.

- **정의 등록** = "어떤 entityType 의 어떤 NUMBER 컬럼을 어떤 방향으로 랭킹화할지" → **JWT**
- **점수 갱신** = 별도 API 없음. **레코드를 저장하면 자동 동기화** → 게임은 그냥 `PUT .../record` 만 하면 됨
- **랭킹 조회** = top-N / 내 순위 → **API 키**

## 1. 정의 관리 (JWT 필요)

### 등록 — `POST /projects/{projectId}/leaderboards`

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| name | string | ✅ | 표시 이름 |
| columnName | string | ✅ | 집계 기준. 해당 scope 의 **NUMBER 스키마**여야 함 |
| order | `ASC`\|`DESC` | ✅ | ASC=작을수록 상위(레이스 타임), DESC=클수록 상위(점수) |
| entityType | string | ❌ | 기본 `user` |
| segment | object | ❌ | 부분 랭킹 조건. 예: `{ "gender": 0 }`. segment 컬럼도 스키마에 있어야 함 |
| resetPolicy | `DAILY`\|`WEEKLY`\|`MONTHLY` | ❌ | 주기 초기화(생략=누적) |
| includeCols | string[] | ❌ | 랭킹 응답에 함께 노출할 컬럼. 예: `["nickname"]` |

```bash
# 전체 best_time 오름차순 + 닉네임 노출
curl -X POST https://api.bbbase.io/projects/{projectId}/leaderboards \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "name": "전체 베스트타임", "columnName": "best_time", "order": "ASC", "includeCols": ["nickname"] }'
```
응답 `data.id` 가 **leaderboardId**. 등록 시 기존 레코드 점수를 **자동 백필**한다.

세그먼트(부분 랭킹)는 `segment` 를 가진 **별도 리더보드**로 등록한다(동적 필터 아님):
```bash
# 여성(gender=0) 전용 best_time 랭킹
-d '{ "name": "여성 베스트타임", "columnName": "best_time", "order": "ASC", "segment": { "gender": 0 } }'
# 조합도 가능: "segment": { "gender": 0, "country": "KR" }
```
레코드가 segment 조건을 **모두 만족할 때만** 그 랭킹에 반영된다. 조건을 더는 만족하지
않게 되면(예: gender 변경) 자동 제외된다.

### 목록 / 단건 / 수정 / 삭제

```bash
GET    /projects/{projectId}/leaderboards
GET    /projects/{projectId}/leaderboards/{id}
PATCH  /projects/{projectId}/leaderboards/{id}   # name/order/resetPolicy/includeCols 만 수정 가능
DELETE /projects/{projectId}/leaderboards/{id}   # score 함께 삭제
```
> 정렬 기준 자체(entityType/columnName/segment)는 수정 불가 — 바꾸려면 새로 등록.

에러: columnName 이 없거나 NUMBER 아님 → `INVALID_LEADERBOARD_COLUMN` /
동일 (entityType, columnName, order, segment) 중복 등록 → `LEADERBOARD_DUPLICATE`.

## 2. 랭킹 조회 (API 키 필요)

`X-API-Key: {API_KEY}` 헤더.

### Top-N — `GET .../leaderboards/{leaderboardId}/ranks?limit=&offset=`

`limit` 기본 50/최대 100, `offset` 기본 0. 정렬방향·노출컬럼은 정의에 귀속되므로
쿼리로 안 보낸다.

```bash
curl "https://api.bbbase.io/projects/{projectId}/leaderboards/{leaderboardId}/ranks?limit=50&offset=0" \
  -H "X-API-Key: {API_KEY}"
```
```json
{ "success": true, "data": {
  "items": [ { "rank": 1, "entityId": "u4", "score": 20, "nickname": "Dave" } ],
  "page": { "limit": 50, "offset": 0, "total": 5 } } }
```
다음 페이지는 `offset` 을 limit 만큼 늘린다.

### 내 순위 — `GET .../leaderboards/{leaderboardId}/ranks/{entityId}`

```json
{ "success": true, "data": {
  "rank": 4, "entityId": "u1", "score": 50, "total": 5, "percentile": 80, "nickname": "Alice" } }
```
동점은 먼저 도달한 쪽이 상위(updatedAt tie-break). 점수 없는 엔티티 →
`LEADERBOARD_SCORE_NOT_FOUND`.

## 3. 점수 갱신은 자동

랭킹 갱신 API는 없다. 레코드 저장이 곧 점수 갱신이다.
```bash
curl -X PUT https://api.bbbase.io/projects/{projectId}/entities/user/u1/record \
  -H "Content-Type: application/json" -H "X-API-Key: {API_KEY}" \
  -d '{ "data": { "best_time": 12.3 } }'
```
컬럼 `compareMode` 가 그대로 적용된다 — `best_time` 이 MIN 이면 **더 빠른 기록일 때만**
순위가 갱신된다.

## 전형적 흐름
1. (1회, JWT) `POST /leaderboards` 로 등록 → id 저장
2. (런타임, API키) 게임은 평소처럼 `PUT .../record` 로 점수 저장 — 랭킹 자동 반영
3. (런타임, API키) `GET .../ranks?limit=N` 로 top-N, `GET .../ranks/{userId}` 로 내 순위 표시
