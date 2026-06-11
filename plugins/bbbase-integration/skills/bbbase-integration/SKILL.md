---
name: bbbase-integration
description: >-
  BBBase(자체 호스팅 게임 BaaS) REST API 연동 스킬. 게임 클라이언트/서버에서 유저
  데이터 저장·불러오기, 리더보드/랭킹, 닉네임·길드명 중복 방지, 일일/주간 리셋,
  스키마 정의, 감사로그, 게임 플레이어 로그인/인증(게스트 로그인)을
  BBBase API로 구현할 때 사용한다. "점수를 서버에 저장", "랭킹 붙여줘", "유저 데이터
  클라우드 저장/동기화", "닉네임 중복 검사", "베스트타임 리더보드",
  "게스트 로그인", "BBBase", "백엔드 연동", "BBBase Unity SDK" 같은 요청이 나오면 추측으로
  엔드포인트를 만들지 말고 반드시 이 스킬을 먼저 참고할 것. Unity 프로젝트에 BBBase SDK
  (`Assets/BBBase`/`using BBBaseSdk`)가 설치돼 있으면 REST 대신 SDK API 로 연동한다.
  인증 헤더 종류, 응답 envelope, 에러코드, compareMode 병합 규칙이 모두 고정 규약이라
  잘못 추측하면 동작하지 않는다.
---

# BBBase 연동

BBBase는 게임 개발자용 자체 호스팅 BaaS다. 게임은 **REST 호출**로 연동한다 —
`fetch`/`UnityWebRequest`/`HttpClient` 등으로 직접 호출하면 된다. 이 스킬은 그 호출
규약을 담는다.

> **⚡ Unity 게임이면 공식 SDK 를 먼저 확인하라.** 프로젝트에 `Assets/BBBase/`(또는
> 코드에 `using BBBaseSdk;` / `BBBase.Init()`)가 보이면 **BBBase Unity SDK 가 설치된 것**이다.
> 그땐 `UnityWebRequest` 코드를 손수 짜지 말고 **SDK API 를 쓰고 `references/unity-sdk.md`
> 를 읽어라**(SDK 가 헤더·envelope·세션을 대신 처리). SDK 가 없거나 고객이 REST 를
> 원하면 아래 REST 규약대로 진행한다. 어느 쪽이든 인증·envelope·compareMode 규약은 동일하다.

## 0. 먼저 자격증명 확보

호출에는 **BASE_URL**, **PROJECT_ID**, **API_KEY** 가 필요하다. 셋의 성격이 다르다:

- **BASE_URL** — 먼저 `BBBase_Keys.md` 나 게임 설정에 BASE_URL 이 있으면 **그 값을 쓴다**(예:
  개발 단계엔 dev 서버 주소). 없으면 **기본값 `https://api.bbbase.io`(prod)** 를 쓴다 — 이 경우
  물어보지 말고 기본값을 적용하라. (SDK 를 쓰면 BASE_URL 은 `BBBaseSettings` 에셋에 있으니
  코드/가이드에서 신경 쓸 필요가 없다 — dev↔prod 는 그 에셋의 `Base Url` 만 바꾼다.)
- **PROJECT_ID · API_KEY** — 프로젝트마다 다른 값. 프로젝트 루트의 `BBBase_Keys.md` 나 게임
  환경설정에서 읽고, 없으면 **개발자에게 직접 물어본다**(추측·공란 금지).
- 아직 BBBase 프로젝트 자체가 없다면(=계정/프로젝트/키 발급 전) → `references/setup.md` 를 읽고
  최초 셋업부터 진행한다. 발급받은 키는 `BBBase_Keys.md` 에 적고 `.gitignore` 에 추가(커밋 금지).

> ℹ️ BBBase 는 BBSofts 내부 테스트용 dev 서버(`http://178.105.162.85:4001`, DB 분리)도 운영하지만,
> 고객 게임은 prod(`https://api.bbbase.io`)를 쓴다. 개발자가 dev 주소를 명시한 경우에만 그걸 쓴다.

> ⚠️ API_KEY 는 게임 클라이언트에 임베드되는 **공개 취급** 키지만, 그래도 소스
> 형상관리에는 커밋하지 않는다(키 로테이션을 쉽게 하기 위해).

## 1. 인증 — 세 종류를 구분하라 (가장 흔한 실수)

호출하는 API에 따라 헤더가 **다르다**. 섞으면 401 이 난다.

| 인증 | 헤더 | 적용 대상 | 누가 호출 |
|---|---|---|---|
| **API 키** | `X-API-Key: {API_KEY}` | 게임 데이터(레코드, 엔티티, **랭킹 조회**) | 게임 클라이언트 |
| **게임유저 토큰** | `Authorization: Bearer {accessToken}` | 레코드 호출 시 API 키와 **함께** — 본인 신원 증명 | 게임 클라이언트(플레이어) |
| **운영자 JWT** | `Authorization: Bearer {accessToken}` | 관리(스키마, 리더보드/유니크 **정의**, 리셋잡, 감사로그, 프로젝트, 키 관리) | 운영자/콘솔/CLI |

핵심 경계선:
- **게임 런타임에서 매번 호출**하는 건 거의 다 **API 키**다 (점수 저장, 기록 조회, 랭킹 보기).
- **셋업/정의/운영**(한 번 또는 가끔)은 **운영자 JWT** 다 (컬럼 정의, 리더보드 등록, 리셋 설정).
- 리더보드는 둘로 갈린다 — **정의 등록은 운영자 JWT**, **랭킹 조회는 API 키**.

> ⚠️ **게임유저 토큰과 운영자 JWT 는 둘 다 `Authorization: Bearer` 지만 전혀 다른 토큰이다.**
> 게임유저 토큰은 플레이어가 **게스트 로그인**(`POST /projects/:pid/auth/guest`)으로 받는다.
> 레코드 호출 때 API 키와 함께 붙인다. 이게 있어야 서버가 "경로의 userId 가
> 진짜 본인인지" 확인해 남의 레코드 조작(`403`)을 막는다. **userId 는 BBBase 가 로그인 시 발급**
> — 직접 만들지 말 것. 게스트 로그인·토큰 첨부·refresh 전체는 `references/game-auth.md` 참고.
> (서버 토글 `GAME_USER_AUTH_REQUIRED` 가 off 면 토큰 없이도 동작하지만, 연동 코드는 지금
> 넣어두는 게 맞다 — off 에서도 무해하고 on 전환 시 그대로 동작.)

운영자 JWT 발급/갱신(로그인, refresh, 만료 처리)은 `references/setup.md` 참고.

## 2. 응답 형식 — 항상 envelope

모든 응답은 아래로 감싸진다.

```json
{ "success": true,  "data": { ... } }
{ "success": false, "error": { "code": "ERROR_CODE", "message": "설명" } }
```

게임 코드는 `error.code` **로 분기**한다. `message` 는 사람용이라 바뀔 수 있으니
조건문에 쓰지 말 것. 실제 데이터는 `data` 안에 있다(레코드는 `data.data` 에 JSONB).

## 명세의 권위 — `/docs-json` (정확한 모양은 여기서)

BBBase 서버는 자기 API 명세를 **OpenAPI(Swagger)** 로 자동 배포한다. 코드에서 자동
생성되므로 **항상 최신**이다.

```
{BASE_URL}/docs        # 브라우저용 인터랙티브 UI
{BASE_URL}/docs-json   # 같은 내용의 raw JSON (에이전트가 읽기 좋음)
```

역할 분담 — 둘은 서로 다른 걸 안다:

- **`/docs-json` = "모양"** : 정확한 경로, 필드명, 타입, 필수 여부. → **이 스킬에 안 적힌
  엔드포인트/필드를 다뤄야 하거나, 이 스킬이 오래돼 보이면 `/docs-json` 을 권위로 삼아라.**
  덕분에 BBBase에 엔드포인트·필드가 추가돼도 이 스킬을 고칠 필요가 없다.
- **이 스킬 = "동작/의미"** : compareMode 병합 규칙, 리더보드 점수 자동 동기화, 유니크
  검사 시점·null 처리, segment 매칭 등 **행동 규칙은 OpenAPI에 없다.** 그건 이 스킬과
  `references/` 가 권위다.

> ⚠️ **`/docs-json` 이 404 면** 그 서버는 Swagger가 꺼진 것(`SWAGGER_DISABLED=1`)이거나
> 아직 배포 안 된 것이다. 이때는 이 스킬의 인라인 내용과 `references/` 로 진행하고,
> 모르는 엔드포인트는 개발자에게 물어본다. (404 라고 게임 데이터 API가 죽은 건 아니다.)

## 3. 가장 흔한 작업 — 유저 레코드 저장/불러오기

유저 1명당 프로젝트 1개에 레코드 1개. 모든 값은 `data`(JSONB) 한 곳에 들어간다.

```bash
# 불러오기
curl https://api.bbbase.io/projects/{PROJECT_ID}/entities/user/{userId}/record \
  -H "X-API-Key: {API_KEY}"

# 저장 (upsert — 신규면 생성, 기존이면 compareMode 규칙으로 병합)
curl -X PUT https://api.bbbase.io/projects/{PROJECT_ID}/entities/user/{userId}/record \
  -H "Content-Type: application/json" -H "X-API-Key: {API_KEY}" \
  -d '{ "data": { "best_time": 4.35, "stars": 120 } }'
```

저장이 **단순 덮어쓰기가 아니라는 점이 핵심**이다. 각 컬럼은 스키마에 정의된
`compareMode` 에 따라 병합된다:

| compareMode | 동작 | 쓰임새 |
|---|---|---|
| `NONE`(기본) | 항상 덮어씀 | 닉네임, 설정값 |
| `MIN` | 더 작을 때만 갱신 | 레이스 타임(`best_time`) |
| `MAX` | 더 클 때만 갱신 | 최고 점수, 최고 스테이지 |
| `INCREMENT` | 기존값 + 보낸값 (원자적, 동시성 안전) | 누적 카운트(플레이 횟수) |

그래서 클라이언트는 "현재 기록이 더 좋은지" 비교할 필요 없이 그냥 PUT 하면 된다 —
서버가 `MIN`/`MAX` 로 막아준다. 동시 저장으로 인한 재화 중복도 서버가 락으로 방지한다.

> 이 컬럼들(`best_time` 등)은 미리 스키마에 정의돼 있어야 한다. 스키마에 없는 컬럼을
> PUT 하면 `UNKNOWN_COLUMN`. 스키마 정의는 `references/records.md` 참고.
>
> 위 예제의 `{userId}` 는 **게스트 로그인이 발급한 값**을 쓴다. 게임유저 인증이 켜진
> 환경에선 이 호출에 `Authorization: Bearer {accessToken}` 도 함께 붙여야 본인 레코드로
> 통과한다(없거나 남의 userId 면 401/403). 게스트 로그인 흐름은 `references/game-auth.md`.

## 4. 작업별 가이드 — 필요한 것만 읽어라

| 하려는 것 | 읽을 파일 |
|---|---|
| **Unity 게임 + SDK 설치됨**(`Assets/BBBase/`) → SDK 로 연동 | `references/unity-sdk.md` |
| 최초 셋업(계정·프로젝트·API키 발급), 운영자 로그인/토큰 갱신 | `references/setup.md` |
| 게임 플레이어 게스트 로그인, 게임유저 토큰 발급·첨부·refresh, 본인 레코드 소유권 | `references/game-auth.md` |
| 컬럼(스키마) 정의·수정, 유저 외 엔티티(길드/그룹/시즌) 레코드 | `references/records.md` |
| 리더보드 등록 + top-N/내 순위 조회 | `references/leaderboard.md` |
| 닉네임·길드명 등 중복 금지 | `references/unique-constraints.md` |
| 주기적 리셋(일/주/월), 변경 이력(감사로그) 조회 | `references/reset-and-audit.md` |

## 4.5 운영자 셋업이 필요한 작업 — 먼저 개발자에게 물어라 (중요)

리더보드 등록, 스키마 컬럼 정의, 유니크 제약 등록, 리셋잡 설정은 **운영자(JWT) 권한**이라
게임 코드만으로는 안 되고 **한 번의 셋업**이 필요하다. "랭킹 붙여줘", "닉네임 중복 막아줘"
같은 요청이 오면 게임 코드(조회/저장)만 짜고 끝내지 말고, **그 셋업을 누가 할지 개발자에게
먼저 물어라.** 두 가지 선택지를 제시한다:

1. **내가(에이전트) CLI 로 직접 추가** — 이 경우 **반빛베이스 운영자 로그인 정보(이메일/
   비밀번호)** 가 필요하다고 명확히 밝혀라. 동의하면 `bbbase auth:login` →
   `bbbase schema:create` / 리더보드·유니크 등록(`references/` 의 해당 curl/CLI)을 직접 실행한다.
   비밀번호는 개발자가 직접 입력하게 하거나(가능하면) `BBBASE_*` 환경변수로 받는다 —
   평문으로 받아 코드/로그에 남기지 말 것.
2. **개발자가 대시보드(`https://bbbase.io`)에서 직접 추가** — 이때는 **무엇을** 만들어야 하는지
   정확히 안내한다(예: "user 스코프에 `best_time` NUMBER MIN 컬럼 추가 → 리더보드 `best_time`
   ASC 등록"). 그리고 에이전트는 **게임 코드(랭킹 조회)만** 작성한다.

> 어느 쪽이든 **PROJECT_ID** 가 필요하다. 운영자 로그인 정보는 게임 API_KEY 와 **다른**
> 자격증명이다(게임 클라용 공개 키 ≠ 운영자 계정). 추측하거나 코드에 하드코딩하지 말 것.

## 5. 자주 만나는 에러코드

| code | 의미 | 보통의 대처 |
|---|---|---|
| `UNKNOWN_COLUMN` | 스키마에 없는 컬럼을 저장 | 먼저 스키마에 컬럼 정의 |
| `DUPLICATE_VALUE` (409) | 유니크 제약 컬럼에 이미 쓰인 값 | 다른 값 요청(닉네임 중복 안내) |
| `RECORD_NOT_FOUND` / `ENTITY_RECORD_NOT_FOUND` | 레코드 없음 | 신규 유저로 처리 |
| `RATE_LIMIT_EXCEEDED` / `TOO_MANY_REQUESTS` | 호출 한도 초과 | 백오프 후 재시도 |
| `LEADERBOARD_SCORE_NOT_FOUND` | 랭킹에 아직 점수 없음 | "기록 없음" 으로 표시 |
| 401 / `UNAUTHORIZED` | 인증 헤더 잘못됨 | API키 vs 게임유저 토큰 vs 운영자 JWT 확인(섹션 1) |
| 403 / `FORBIDDEN` | 남의 userId 로 레코드 접근(인증 켜진 경우) | 경로 userId 를 로그인 응답의 userId 로 교정(`references/game-auth.md`) |

Rate limit: 게임 데이터 API는 **API 키당 분당 600회**, 인증 API는 IP당 분당 10회.
요청 본문은 최대 256KB(게임 레코드는 보통 수 KB라 문제없음).

## 작업 원칙

- 엔드포인트 경로·필드명을 **추측하지 말 것.** 우선순위: ① 이 스킬/레퍼런스 →
  ② `GET {BASE_URL}/docs-json`(라이브 OpenAPI, 가장 정확) → ③ 그래도 모르면 개발자에게
  질문. `/docs-json` 이 404 면 Swagger 비활성 상태이니 ①과 ③으로만 진행한다.
- 게임 코드에 BBBase 호출을 넣을 때는 응답을 `success` 로 분기하고 `error.code` 별
  처리를 넣는다 — 네트워크/서버 오류로 게임이 죽지 않게.
- API_KEY/토큰을 코드에 하드코딩하지 말고 설정/환경값으로 분리한다.
- **(Windows 한정) curl 로 한글 등 비ASCII 데이터를 보낼 때 인라인 `-d '{"name":"한글"}'`
  를 쓰지 말 것** — 셸 코드페이지 때문에 인코딩이 깨져 DB에 깨진 값이 저장된다(리더보드
  표시이름, 닉네임 등). **한글 값은 그대로 유지하되**, UTF-8 파일로 저장 후
  `curl --data @body.json -H "Content-Type: application/json; charset=utf-8"` 로 보내거나
  BBBase 대시보드 UI로 입력한다. 한글을 영어로 바꾸라는 뜻이 아니라 **전송 방식만** 바꾸는
  것이다. (게임 소스코드·주석의 한글은 무관 — 런타임 UTF-8 이라 정상. 컬럼명은 어차피
  `^[a-z][a-z0-9_]*$` 라 항상 영어다.)
