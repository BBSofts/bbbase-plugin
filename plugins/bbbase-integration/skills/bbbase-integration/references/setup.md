# 최초 셋업 + 토큰 관리

BBBase 프로젝트가 아직 없을 때 **최초 1회** 수행한다. 완료하면 `PROJECT_ID` 와
`API_KEY` 가 확보되고, 이후 게임은 그 두 값만으로 동작한다.

```
BASE_URL : https://api.bbbase.io   (개발 단계엔 BBBase_Keys.md 에 적힌 dev 주소)
```

> 정확한 경로·필드·타입은 `{BASE_URL}/docs-json`(라이브 OpenAPI)이 권위다. 아래는 그 위에
> **순서·의미**를 더한 것 — 어긋나면 `/docs-json` 을 믿어라. (404 면 Swagger 비활성.)

## 셋업 순서

```
1. 계정 생성(회원가입)  → tenantId
2. 로그인               → accessToken(1h) / refreshToken(30d)
3. 프로젝트 생성        → projectId         (JWT 필요)
4. API 키 발급          → key (1회만 노출!)  (JWT 필요)
```

### 1. 계정 생성 — `POST /tenants`

| 필드 | 필수 | 설명 |
|---|---|---|
| email | ✅ | 중복 불가 |
| name | ✅ | 스튜디오명 |
| password | ✅ | 8자 이상 |

```bash
curl -X POST https://api.bbbase.io/tenants \
  -H "Content-Type: application/json" \
  -d '{ "email": "admin@mygame.com", "name": "My Game Studio", "password": "secret1234" }'
```
응답 `data.id` 가 **tenantId**.

### 2. 로그인 — `POST /auth/login`

```bash
curl -X POST https://api.bbbase.io/auth/login \
  -H "Content-Type: application/json" \
  -d '{ "email": "admin@mygame.com", "password": "secret1234" }'
```
응답에 `accessToken`(1시간), `refreshToken`(30일). 이후 관리 API는
`Authorization: Bearer {accessToken}` 헤더 사용.

> 본인 계정 확인: `GET /tenants/me` (JWT). 전체 테넌트 목록 API는 없음(보안).

### 3. 프로젝트 생성 — `POST /projects` (JWT)

| 필드 | 필수 | 설명 |
|---|---|---|
| name | ✅ | 프로젝트명 |
| description | ❌ | 설명 |

```bash
curl -X POST https://api.bbbase.io/projects \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "name": "My Puzzle Game", "description": "하이퍼캐주얼 퍼즐" }'
```
응답 `data.id` 가 **projectId**. (내 목록: `GET /projects`)

### 4. API 키 발급 — `POST /projects/{projectId}/api-keys` (JWT)

```bash
curl -X POST https://api.bbbase.io/projects/{projectId}/api-keys \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "name": "game-client-prod" }'
```
응답 `data.key` 가 실제 API 키. **이 응답에서 단 1회만 노출**된다(DB에는 해시만 저장).

> ⚠️ 키를 받은 즉시:
> 1. 프로젝트 루트에 `BBBase_Keys.md` 생성하고 `Project ID`, `Key ID`, `Prefix`, `API Key` 기록.
> 2. `.gitignore` 에 `BBBase_Keys.md` 추가(SVN은 `svn:ignore`, Perforce는 `.p4ignore`).
>
> `BBBase_Keys.md` 예시:
> ```markdown
> # BBBase API Keys
> ## game-client-prod (발급일: 2026-05-30)
> - Project ID : {projectId}
> - Key ID     : {id}
> - Prefix     : {prefix}
> - API Key    : {key}
> ```

키 목록: `GET /projects/{projectId}/api-keys` (해시만, 원본 키는 안 보임).
키 폐기: `DELETE /projects/{projectId}/api-keys/{keyId}` (즉시 인증 거부).

## 토큰 갱신/만료 처리

accessToken 은 1시간이라 만료되면 401 이 난다. 그때 refresh 한다.

```bash
# 갱신 — POST /auth/refresh
curl -X POST https://api.bbbase.io/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{ "refreshToken": "{refreshToken}" }'
```
응답에 **새** accessToken + **새** refreshToken. ⚠️ 갱신 시 기존 refreshToken 은
즉시 무효화(rotation)되므로 응답의 새 토큰으로 **교체 저장**해야 한다.

```bash
# 로그아웃 — POST /auth/logout  { refreshToken }
# 비밀번호 변경 — POST /auth/change-password (JWT) { currentPassword, newPassword(8자+) }
#   → 변경 시 그 계정의 모든 refreshToken 무효화, 재로그인 필요
```

## CLI 대안

oclif 기반 `bbbase` CLI 도 있다(로그인 토큰을 `~/.bbbase/credentials.json` 에 자동 보관):

```bash
bbbase tenant:create --email admin@mygame.com --name "My Game Studio" --password secret1234
bbbase auth:login   --email admin@mygame.com --password secret1234
bbbase project:create --name "My Puzzle Game"
```
