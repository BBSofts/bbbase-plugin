# 공용 Config (필수 업데이트·원격 플래그·서버 튜닝값)

```
BASE_URL : https://api.bbbase.io
```

> 정확한 경로·필드·타입은 `{BASE_URL}/docs-json`(라이브 OpenAPI)이 권위다. 아래는
> 그 위에 **동작·의미·함정**을 더한 것 — 어긋나면 `/docs-json` 을 믿어라.

## 개념 — 프로젝트 전역 공용 설정값

게임 클라가 **API 키만으로**(게임유저 토큰 불필요, **로그인 전에도**) 읽는 key→value(JSON)
저장소. 전 유저가 공유하는 설정을 서버에서 원격으로 바꾼다. 대표 용도:

- **필수 업데이트** — 최소 요구 버전을 서버에 두고, 게임 시작 시 받아 현재 버전과 비교 → 낮으면
  스토어로 유도하고 진입 차단.
- **원격 기능 플래그** — 이벤트/신기능 on/off 를 앱 재배포 없이 토글.
- **서버 튜닝값** — 골드 배율, 점검 공지 문구 같은 밸런스/운영 상수.

경계선(헷갈리지 마라):
- **Config** = 운영자가 쓰고, 클라가 **읽는다**(읽기 전용). 전역 공유. ↔ **유저 레코드**는 유저
  개인 데이터(유저 소유, 게임유저 토큰 필요, compareMode 병합).
- **Config** = 작은 **설정값**(JSON, ≤32KB). 이미지·사운드·에셋번들 같은 **파일은 담지 마라**
  (그건 오브젝트 스토리지의 영역 — Config 아님).
- **Config** ↔ **로그 수집**: 로그는 클라가 *쓰는* append-only 이벤트. Config 는 반대(운영자가 씀).

## 1. 값 읽기 (API 키 — 게임 클라)

### `GET /projects/{projectId}/configs/{key}`

`X-API-Key` 만 필요(게임유저 토큰 불필요). 그래서 **로그인 화면 이전**에 호출 가능.
응답은 서버에서 5분 캐시된다.

```bash
curl "https://api.bbbase.io/projects/{PROJECT_ID}/configs/force_update" \
  -H "X-API-Key: {API_KEY}"
# → { "success": true, "data": {
#       "key": "force_update",
#       "value": { "minVersion": "1.2.0", "storeUrl": "https://...", "message": "업데이트가 필요합니다" },
#       "updatedAt": "2026-07-11T..." } }
```

- 실제 설정은 `data.value` 안에 있다(임의 JSON — 객체/배열/원시값).
- 키가 없으면 `CONFIG_NOT_FOUND`(404). 게임은 이를 **"설정 없음 → 기본 동작"**으로 처리하고,
  Config 조회 실패(네트워크 오류 포함)로 게임이 죽지 않게 하라(실패 시 기존 진입 허용 등).

## 2. 값 관리 (운영자 JWT)

```bash
# 저장(upsert) — value 는 임의 JSON
curl -X PUT "https://api.bbbase.io/projects/{PROJECT_ID}/configs/force_update" \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "value": { "minVersion": "1.2.0", "storeUrl": "https://...", "message": "업데이트가 필요합니다" } }'

# 목록 / 삭제
curl "https://api.bbbase.io/projects/{PROJECT_ID}/configs" -H "Authorization: Bearer {accessToken}"
curl -X DELETE "https://api.bbbase.io/projects/{PROJECT_ID}/configs/force_update" -H "Authorization: Bearer {accessToken}"

# CLI
bbbase config:set {PROJECT_ID} force_update --value '{"minVersion":"1.2.0","storeUrl":"https://...","message":"..."}'
bbbase config:list {PROJECT_ID}
bbbase config:remove {PROJECT_ID} force_update
```

- 키 형식: `A-Z a-z 0-9 . _ - :` 1~128자. 값은 직렬화 32KB 이하.
- 대시보드 프로젝트 → **공용 Config** 화면에서도 추가·편집·삭제할 수 있다.
- **반영 지연**: 공개 읽기는 5분 캐시라 값 변경이 최대 5분 뒤 게임에 반영된다.

## 3. 필수 업데이트 구현 흐름 (대표 예시)

1. (운영자, 1회) `force_update` 키에 최소 버전 등 저장 — 대시보드/CLI/REST 중 택1.
2. (게임 런타임) **로그인/게임 진입 전** `GET .../configs/force_update` 호출(API 키만).
3. `data.value.minVersion` 과 현재 앱 버전을 비교 → 낮으면 업데이트 팝업(스토어 링크) 후 진입 차단.
   `CONFIG_NOT_FOUND` 나 네트워크 오류면 **차단하지 말고** 정상 진입(운영 실수로 전 유저 잠금 방지).

> ⚠️ **운영자 셋업이 필요하다.** 값 저장은 운영자(JWT) 권한이다. "필수 업데이트 붙여줘" 요청이 오면
> 게임 코드(조회·비교·팝업)만 짜고 끝내지 말고, **누가 `force_update` 값을 넣을지** 개발자에게
> 물어라 — ① 에이전트가 CLI/REST 로(→ 운영자 로그인 정보 필요), 또는 ② 개발자가 대시보드에서.

## SDK 사용 (설치돼 있으면 REST 대신)

- **Unity**: `var f = await BBBase.Config.GetAsync<ForceUpdate>("force_update");`(없으면 null).
  원본은 `BBBase.Config.GetRawAsync(key)`.
- **Godot**: `var res := await BBBase.config.get_value("force_update")` → `res.data.value`.
  간편히 값만: `BBBase.config.get_value_or("force_update", default)`.

## 에러코드

| code | 의미 | 대처 |
|---|---|---|
| `CONFIG_NOT_FOUND` (404) | 그 키 없음 | "설정 없음=기본 동작"으로 처리 |
| `INVALID_API_KEY` (401) | 공개 읽기에 X-API-Key 누락/무효 | 헤더 확인 |
| `INVALID_INPUT` (400) | 키 형식 위반 또는 값 32KB 초과 | 키/값 교정 |
| `FORBIDDEN` (403) | 남의 프로젝트에 쓰기/삭제(운영자) | 프로젝트 소유 확인 |
