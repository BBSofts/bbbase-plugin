# 유니크 제약 (닉네임·길드명 등 중복 금지)

```
BASE_URL : https://api.bbbase.io
```

> 정확한 경로·필드·타입은 `{BASE_URL}/docs-json`(라이브 OpenAPI)이 권위다. 아래 표는
> 그 위에 **동작·의미·함정**을 더한 것 — 표가 라이브와 어긋나면 `/docs-json` 을 믿어라.
> (`/docs-json` 이 404 면 Swagger 비활성이니 이 문서를 사용한다.)

## 개념 — 사전 등록형

"프로젝트 안에서 값이 겹치면 안 되는 컬럼"을 **미리 등록**하면, 이후 레코드 저장 시
중복이 **DB 레벨에서** 차단된다.

- **정의 등록** = "어떤 entityType 의 어떤 컬럼을 중복금지로 둘지" → **JWT**
- **검사** = 별도 API 없음. 등록된 컬럼은 **레코드 저장 시점에 자동 검사** → 중복이면 **409 `DUPLICATE_VALUE`**

동작 특성:
- **대소문자 구분** — `"Alice"` 와 `"alice"` 는 다른 값.
- **null 은 검사 제외** — 값이 없는(null/미설정) 레코드는 중복 대상이 아님.
- **동시 저장 안전** — 두 클라이언트가 같은 닉네임을 따닥 저장해도 한쪽만 통과.

## 1. 정의 관리 (JWT 필요)

### 등록 — `POST /projects/{projectId}/unique-constraints`

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| columnName | string | ✅ | 중복금지 컬럼. 해당 scope 스키마에 존재 + **STRING/NUMBER/BOOLEAN** 만(ARRAY/OBJECT 불가) |
| entityType | string | ❌ | 기본 `user` |

```bash
# user 의 nickname 중복금지
curl -X POST https://api.bbbase.io/projects/{projectId}/unique-constraints \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "columnName": "nickname" }'

# guild 의 guildName 중복금지
curl -X POST https://api.bbbase.io/projects/{projectId}/unique-constraints \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "entityType": "guild", "columnName": "guildName" }'
```
등록 시 기존 레코드 값을 **1회 백필**한다. 이미 중복이 있었다면 그 행은 조용히
건너뛰고(등록은 성공), 건너뛴 개수를 응답 `skippedDuplicates` 로 알려준다 — 즉 기존
중복은 그대로 두고 **신규 저장부터** 차단하는 정책.

에러: columnName 이 스키마에 없거나 ARRAY/OBJECT → `INVALID_UNIQUE_COLUMN` /
동일 (entityType, columnName) 중복 등록 → `UNIQUE_CONSTRAINT_DUPLICATE`.

### 목록 / 단건 / 삭제

```bash
GET    /projects/{projectId}/unique-constraints
GET    /projects/{projectId}/unique-constraints/{id}
DELETE /projects/{projectId}/unique-constraints/{id}   # 쌓인 값도 함께 정리
```
> 수정(PATCH) API 없음 — 바꾸려면 삭제 후 재등록. 없는 id → `UNIQUE_CONSTRAINT_NOT_FOUND`.

## 2. 중복 차단 (API 키 — 레코드 저장 시 자동)

```bash
# u1 이 "Alice" 로 저장 → 성공
curl -X PUT https://api.bbbase.io/projects/{projectId}/entities/user/u1/record \
  -H "Content-Type: application/json" -H "X-API-Key: {API_KEY}" \
  -d '{ "data": { "nickname": "Alice" } }'

# u2 가 같은 "Alice" → 409 DUPLICATE_VALUE
```
```json
{ "success": false, "error": {
  "code": "DUPLICATE_VALUE",
  "message": "Value \"Alice\" for column \"nickname\" is already in use" } }
```

- **값 변경**(닉네임 변경) — 같은 엔티티가 값을 바꾸면 옛 값 예약 해제, 새 값 등록. 비워진 옛 값은 다른 엔티티가 다시 쓸 수 있음.
- **값 제거** — 컬럼을 null/미설정으로 저장하면 예약 해제.
- 이 검사는 upsert 재시도 루프와 무관하게 **즉시 실패**(재시도해도 영원히 중복이므로).

## 게임 연동 흐름
1. (1회, JWT) `nickname` 컬럼이 스키마에 없으면 먼저 `POST /schemas` 로 `{ columnName: "nickname", type: "STRING" }` 생성
2. (1회, JWT) `POST /unique-constraints` 로 `{ "columnName": "nickname" }` 등록
3. (런타임) 닉네임 설정 화면에서 `PUT .../record` 응답이 `DUPLICATE_VALUE` 면 "이미 사용 중인 닉네임" 안내
