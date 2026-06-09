# 스키마 정의 + 레코드/엔티티 데이터

```
BASE_URL : https://api.bbbase.io
```

> 정확한 경로·필드·타입은 `{BASE_URL}/docs-json`(라이브 OpenAPI)이 권위다. 아래 표는
> 그 위에 **동작·의미·함정**을 더한 것 — 표가 라이브와 어긋나면 `/docs-json` 을 믿어라.
> (`/docs-json` 이 404 면 Swagger 비활성이니 이 문서를 사용한다.)

## 데이터 모델 핵심

게임 데이터는 고객이 만든 테이블이 아니라 **메타 스키마 + JSONB** 로 저장된다.

- **스키마(GameSchema)** — "어떤 컬럼이 어떤 타입·병합규칙을 갖는지" 정의하는 메타데이터.
- **레코드(EntityRecord)** — 실제 게임 데이터. `data`(JSONB) 한 칸에 모든 값. `(projectId, entityType, entityId)` 로 유일.
- **entityType** — 자유 문자열. `user`, `group`, `guild`, `season` 등. 새 종류 추가에 **서버 코드 변경 불필요**.

런타임 DDL(테이블 생성 등)은 거부된다. 모든 컬럼은 스키마 메타데이터로만 등록한다.

## 스키마 API (JWT 필요)

`Authorization: Bearer {accessToken}` 헤더 필수.

### 컬럼 생성 — `POST /projects/{projectId}/schemas`

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| columnName | string | ✅ | 영소문자/숫자/밑줄만 |
| type | `STRING`\|`NUMBER`\|`BOOLEAN`\|`ARRAY`\|`OBJECT` | ✅ | 데이터 타입 |
| compareMode | `NONE`\|`MIN`\|`MAX`\|`INCREMENT` | ❌ | 갱신 규칙(기본 NONE=덮어씀) |
| scope | string | ❌ | 엔티티 종류(기본 `user`). 이 값이 곧 entityType |
| defaultValue | any | ❌ | 신규 레코드 초기값(리셋 기준값으로도 활용) |
| nullable | boolean | ❌ | null 허용(기본 false) |

```bash
# best_time: 작을수록 좋은 레이스 타임
curl -X POST https://api.bbbase.io/projects/{projectId}/schemas \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "columnName": "best_time", "type": "NUMBER", "compareMode": "MIN", "defaultValue": 999999 }'

# 그룹(scope=group)의 시도 횟수: 누적 카운트
curl -X POST https://api.bbbase.io/projects/{projectId}/schemas \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "columnName": "attempts", "type": "NUMBER", "compareMode": "INCREMENT", "scope": "group", "defaultValue": 0 }'
```

### 목록 / 수정 / 삭제

```bash
GET    /projects/{projectId}/schemas
GET    /projects/{projectId}/schemas/{schemaId}
PATCH  /projects/{projectId}/schemas/{schemaId}   # 변경할 필드만. columnName 변경 불가
DELETE /projects/{projectId}/schemas/{schemaId}
```

## compareMode 동작 상세

레코드 저장(`PUT .../record`)은 컬럼별 compareMode 로 병합된다:

- **NONE** — 보낸 값으로 항상 덮어씀.
- **MIN** — 보낸 값이 기존값보다 **작을 때만** 갱신. (예: `best_time` 4.5 저장 후 4.8 보내면 무시, 4.2 보내면 갱신)
- **MAX** — 보낸 값이 기존값보다 **클 때만** 갱신. (최고 점수)
- **INCREMENT** — 기존값에 보낸 값을 **더함**(덮어쓰기 아님). 동시 호출에도 원자적으로 누적돼 카운트 유실 없음.

그래서 게임 클라이언트는 비교 로직 없이 그냥 PUT 한다 — 서버가 막아준다.

## 레코드 엔드포인트는 하나 — entityType 으로 구분 (API 키 필요)

모든 레코드는 **단일 범용 엔드포인트** `/projects/{projectId}/entities/{entityType}/{entityId}/record`
로 다룬다(구 `/users/...` 편의 API 는 제거됨). `X-API-Key: {API_KEY}` 헤더 필요.

> ⚠️ **플레이어 본인 데이터는 `entityType=user` 를 쓴다.** `entityType==='user'` 일 때만 서버가
> 소유권(경로 entityId == 로그인 토큰의 userId)을 강제해 남의 레코드 조작을 막는다(`game-auth.md`).
> 다른 이름(`player` 등)을 쓰면 소유권 보호가 사라진다. 길드·시즌 등은 자유 문자열.

```bash
# 유저(플레이어) 레코드 — entityType=user
GET    /projects/{projectId}/entities/user/{userId}/record
PUT    /projects/{projectId}/entities/user/{userId}/record   # upsert (compareMode 병합)
DELETE /projects/{projectId}/entities/user/{userId}/record
```
```bash
curl -X PUT https://api.bbbase.io/projects/{projectId}/entities/user/u1/record \
  -H "Content-Type: application/json" -H "X-API-Key: {API_KEY}" \
  -d '{ "data": { "best_time": 4.35, "stars": 120 } }'
```
조회 응답에서 실제 값은 `data.data` 안에 있다.

## 범용 엔티티 레코드 (API 키 필요)

유저 외 그룹·길드·시즌 등. `entityType` 은 스키마의 `scope` 와 일치해야 한다.

```bash
GET    /projects/{projectId}/entities/{entityType}/{entityId}/record
PUT    /projects/{projectId}/entities/{entityType}/{entityId}/record
DELETE /projects/{projectId}/entities/{entityType}/{entityId}/record
```
```bash
# 그룹 room_42 시도 +1 (INCREMENT)
curl -X PUT https://api.bbbase.io/projects/{projectId}/entities/group/room_42/record \
  -H "Content-Type: application/json" -H "X-API-Key: {API_KEY}" \
  -d '{ "data": { "attempts": 1 } }'

# 길드 최고점수 갱신 (MAX)
curl -X PUT https://api.bbbase.io/projects/{projectId}/entities/guild/guild_001/record \
  -H "Content-Type: application/json" -H "X-API-Key: {API_KEY}" \
  -d '{ "data": { "total_score": 3000 } }'
```

운영자용 레코드 목록(JWT, 커서 페이지네이션):
```bash
GET /projects/{projectId}/entities/{entityType}/records?limit=&cursor=&entityId=
```

## 에러
- 스키마에 없는 컬럼 PUT → `UNKNOWN_COLUMN`
- 레코드 없음 → `RECORD_NOT_FOUND` / `ENTITY_RECORD_NOT_FOUND`
- 유니크 제약 위반 → `DUPLICATE_VALUE` (409, `unique-constraints.md` 참고)
