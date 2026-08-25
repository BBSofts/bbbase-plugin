# 게임 유저 인증 (게스트 로그인)

게임 **플레이어**의 신원을 증명하는 인증이다. 운영자(테넌트) JWT(`setup.md`)와는
완전히 별개다 — 그건 콘솔/CLI용이고, 이건 게임 클라이언트 런타임용이다.

```
BASE_URL : https://api.bbbase.io   (dev: 4001)
```

> 정확한 경로·필드는 `{BASE_URL}/docs-json`(라이브 OpenAPI)이 권위다. 아래는 그 위에
> **순서·의미·함정**을 더한 것. (404 면 Swagger 비활성이니 이 문서로 진행.)

## 왜 필요한가 — 두 토큰 모델

API 키만으로는 "정상 게임 클라이언트"인지까지만 증명된다. API 키는 클라에 임베드되는
**공개 취급** 값이라, 이것만 믿으면 누군가 키를 추출해 **다른 유저의 userId 로 레코드를
조작**할 수 있다. 그래서 레코드 호출은 두 개를 함께 쓴다:

| | 헤더 | 증명하는 것 |
|---|---|---|
| **API 키** | `X-API-Key: {API_KEY}` | "정상 게임 클라이언트다"(프로젝트 식별) |
| **게임유저 토큰** | `Authorization: Bearer {accessToken}` | "이 요청을 보낸 게 진짜 그 유저다"(신원) |

게임유저 토큰이 붙으면 서버가 **토큰 속 userId 와 경로의 `entityId`(entityType=user) 가
같은지** 확인하고, 다르면 `403 FORBIDDEN` 으로 막는다 → 본인 레코드만 읽기/쓰기 가능.

## userId 는 BBBase 가 발급한다 (직접 만들지 말 것)

게스트 로그인을 하면 BBBase 가 그 유저의 `userId`(uuid)를 **발급**해 돌려준다.
**레코드 경로의 `{userId}` 에는 반드시 이 발급된 값을 쓴다.** 기기 ID·임의 문자열을
직접 userId 로 쓰면, 인증이 켜진 환경에서 `403` 이 난다(토큰 userId 와 불일치).

## 1. 게스트 로그인 — `POST /projects/{projectId}/auth/guest` (API 키)

같은 `deviceId` 로 다시 로그인하면 **같은 계정(userId)** 이 반환된다(멱등).

```bash
curl -X POST https://api.bbbase.io/projects/{PROJECT_ID}/auth/guest \
  -H "Content-Type: application/json" -H "X-API-Key: {API_KEY}" \
  -d '{ "deviceId": "{기기_고유_식별자}" }'
```

응답 `data`:

```json
{ "userId": "fc54e345-...", "accessToken": "eyJ...", "refreshToken": "a1b2..." }
```

- `userId` — 이후 모든 레코드 경로에 쓸 식별자. 클라에 저장.
- `accessToken` — 레코드 호출 시 `Authorization: Bearer` 로 첨부. **1시간** 만료.
- `refreshToken` — access 만료 시 갱신용. 안전히 저장(기기 보안 저장소 권장).

> `deviceId` 는 기기마다 **안정적이고 고유**해야 한다(예: 플랫폼 광고ID 대체값, 설치 시
> 생성해 영구 저장한 UUID). 게스트만 쓰면 deviceId 를 잃을 때 진행도도 잃으니, 진짜 보존이
> 필요하면 **소셜 계정을 링킹**해두라(아래 4번). 링킹하면 어느 기기에서든 복구된다.

## 2. 토큰을 붙여 레코드 호출

로그인 후엔 **API 키 + Bearer 토큰을 함께** 보내고, 경로엔 발급받은 userId 를 쓴다.

```bash
# 저장 (본인 레코드)
curl -X PUT https://api.bbbase.io/projects/{PROJECT_ID}/entities/user/{userId}/record \
  -H "Content-Type: application/json" \
  -H "X-API-Key: {API_KEY}" \
  -H "Authorization: Bearer {accessToken}" \
  -d '{ "data": { "best_time": 4.35 } }'
```

다른 유저의 `userId` 로 호출하면 `403 FORBIDDEN`("You can only access your own record").
`user` 외 엔티티(`group`/`guild` 등)는 phase 1 에서 소유권 검사가 없다(로그인만 요구).

## 3. 토큰 갱신 / 로그아웃

```bash
# 갱신 — POST /projects/{projectId}/auth/refresh   { refreshToken }
#   → 새 accessToken + 새 refreshToken. 기존 refreshToken 은 즉시 무효(rotation).
#     반드시 응답의 새 토큰으로 교체 저장. 옛 토큰 재사용은 401.
curl -X POST https://api.bbbase.io/projects/{PROJECT_ID}/auth/refresh \
  -H "Content-Type: application/json" -H "X-API-Key: {API_KEY}" \
  -d '{ "refreshToken": "{refreshToken}" }'

# 로그아웃 — POST /projects/{projectId}/auth/logout  { refreshToken }
#   → 해당 refreshToken 무효화.
```

권장 흐름: 게임 시작 시 저장된 토큰으로 호출 → `401` 이면 refresh → refresh 도 실패면
저장된 `deviceId` 로 게스트 로그인 재수행(같은 userId 복구).

## 4. 계정 링킹 + 클라우드 세이브

클라우드 세이브는 **이미 동작한다** — 레코드가 처음부터 `userId` 로 서버에 저장되니, 같은
계정으로 로그인하면 어느 기기서든 불러온다. 링킹은 **게스트 진행도를 잃지 않고 소셜 계정에
묶는** 기능이다. **핵심: 링킹해도 `userId` 는 안 바뀐다** → 세이브가 그대로 따라온다.

```
POST   /projects/{pid}/auth/link            (X-API-Key + Bearer)  { "provider": "GOOGLE", "idToken": "..." }
DELETE /projects/{pid}/auth/link/{provider}  (X-API-Key + Bearer)  provider = GUEST|GOOGLE|APPS_IN_TOSS
GET    /projects/{pid}/auth/me              (X-API-Key + Bearer)  → { userId, isGuest, providers:[...] }
```

- **게스트→구글 연동**: 게스트 로그인 상태에서 `POST /auth/link {provider:"GOOGLE", idToken}`.
  링킹 바디는 provider 별로 `idToken`(구글) / `authorizationCode`(앱인토스) / `deviceId`(게스트).
- **기기 변경 복구**: 새 기기에선 그냥 **구글 로그인**(`/auth/google`) — 링킹이 아니라 로그인이다.
  그 구글이 가리키는 기존 계정으로 들어가 세이브가 복구된다.
- **링크 해제**: `DELETE /auth/link/{provider}`. **마지막 수단은 해제 불가**(`CANNOT_UNLINK_LAST`).

> ⚠️ `409 IDENTITY_ALREADY_LINKED` — 링크하려는 소셜이 **이미 다른 계정**에 묶였을 때. 서버는
> 자동 머지하지 않고 거부하며 `error.details.conflictUserId` 로 상대 계정을 준다. "기존 계정으로
> 전환?(현재 진행도 사라짐)" 을 유저에게 묻고, 전환 택하면 그 소셜로 **로그인**해 넘어간다.

SDK: Unity `LinkGoogleAsync/UnlinkAsync/GetMeAsync`, Godot `link_google/unlink/get_me`.

## 5. 롤아웃 토글 — `GAME_USER_AUTH_REQUIRED` (서버측)

서버에 이 인증의 **강제 여부** 토글이 있다(운영자 env):

- `true`  → 레코드 호출에 게임유저 토큰 **필수**, 본인 레코드만 허용(위 규칙 적용).
- `false` → 토큰 검사를 건너뜀. 토큰 없이도 기존처럼 동작(레거시 호환).

**게임 클라가 게스트 로그인 연동을 마치기 전까지 `false` 로 두어** 기존 게임이 깨지지
않게 한다. 클라가 로그인+토큰 첨부를 적용해 배포되면 운영자가 `true` 로 전환 → 사칭
차단이 실제로 활성화된다. **연동 코드는 토글과 무관하게 지금 넣어두면 된다**(토글 off
에서도 토큰을 보내는 건 무해하고, on 전환 시 그대로 동작).

## 6. 제재된 계정 처리 — `USER_BANNED`

운영자가 어뷰저를 제재하면 서버가 **모든 요청을 `403 USER_BANNED` 로 거절**한다. 로그인·토큰
갱신도 거부되므로 재로그인으로 빠져나갈 수 없다.

```jsonc
// 어떤 요청이든 제재 중이면
{ "success": false, "error": {
    "code": "USER_BANNED",
    "message": "이 계정은 일시 제재되었습니다",
    "details": { "expiresAt": "2026-08-27T06:22:18.435Z", "reason": "랭킹 점수 조작" }
} }
// expiresAt 이 null 이면 영구 제재
```

**게임이 해야 할 일:** 플레이를 중단하고 정지 안내 화면을 띄운다. `details.expiresAt`(null=영구)과
`details.reason` 을 그대로 보여주면 된다. **재시도 루프를 돌리지 말 것** — 제재가 풀릴 때까지
계속 403 이고 서버만 두드리게 된다.

> 제재 집행은 서버가 한다. 클라가 이 응답을 무시하고 게임을 계속 돌려도 저장·랭킹·보상은 이미
> 전부 막혀 있으므로 **데이터는 안전하다** — 화면 처리는 유저 경험 문제지 보안 문제가 아니다.

`GET /projects/{projectId}/auth/me` 는 제재 중에도 열려 있다(계정 상태 화면을 그릴 수 있게).

공식 SDK 를 쓰면 이 감지가 내장돼 있다 — Godot 은 `BBBase.banned(expires_at, reason)` 시그널,
Unity 는 `BBBase.Banned` 이벤트로 받는다(동시 요청이 여러 번 403 을 받아도 1회만 발생).
자세한 건 `references/godot-sdk.md` / `references/unity-sdk.md`.

## 자주 만나는 에러

| code / status | 의미 | 대처 |
|---|---|---|
| `403` `FORBIDDEN` | 남의 userId 로 레코드 접근 | 경로 userId 를 로그인 응답의 userId 로 교정 |
| `403` `USER_BANNED` | 운영자가 제재한 계정 | 재시도·재로그인 금지. 정지 안내 표시(위 섹션 6) |
| `401` `UNAUTHORIZED` (토큰) | 토큰 누락/만료/위조 | refresh → 실패 시 게스트 재로그인 |
| `401` `INVALID_API_KEY` | API 키 문제 | `X-API-Key` 확인(게임유저 토큰과 별개) |
