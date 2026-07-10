# 로그 수집 (로그인 실패 등 클라 이벤트)

```
BASE_URL : https://api.bbbase.io
```

> 정확한 경로·필드·타입은 `{BASE_URL}/docs-json`(라이브 OpenAPI)이 권위다. 아래는
> 그 위에 **동작·의미·함정**을 더한 것 — 어긋나면 `/docs-json` 을 믿어라.

## 개념 — 게임유저 토큰 없이 쌓는 로그

게임 클라가 **API 키만으로**(게임유저 토큰 불필요) 임의 이벤트 로그를 서버에 쌓는 채널.
주 용도는 **로그인 실패처럼 "인증 전/실패 시점" 이벤트** — 로그인이 안 됐으면 게임유저
토큰이 없어서 일반 레코드 API(`PUT .../record`)로는 못 남긴다(그건 토큰이 필수). 로그
채널이 그 공백을 메운다.

- **수집** = `POST /projects/:pid/logs` → **API 키만** (`X-API-Key`). 게임유저 토큰 불필요.
- **조회** = `GET /projects/:pid/logs` → **운영자 JWT**. 대시보드 "로그" 화면 / CLI `log:list`.

> ⚠️ **신뢰할 수 없는 제보다.** API 키는 게임 빌드에 박혀 배포되는 공개 식별자라, 이 로그는
> 누구나 보낼 수 있다. 게임 상태·과금에 반영하지 말고 **디버깅·통계 참고용**으로만 써라.

## 1. 로그 보내기 (API 키 — 게임 클라)

### `POST /projects/{projectId}/logs`

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| level | `debug`\|`info`\|`warn`\|`error` | ❌ | 생략 시 `info`. 그 외 값은 400 |
| category | string(≤64) | ❌ | 그룹핑 키. 예: `login_fail`, `purchase_error` |
| message | string(≤2000) | ❌ | 사람이 읽는 로그 메시지 |
| platform | string(≤32) | ❌ | 예: `android`, `ios`, `web`, `editor` |
| data | object(≤8KB) | ❌ | 자유 커스텀 필드(실패 사유/코드/스택 등) |

```bash
# 소셜 로그인 실패를 로그로 — 유저 토큰 없이 API 키만
curl -X POST https://api.bbbase.io/projects/{projectId}/logs \
  -H "Content-Type: application/json" -H "X-API-Key: {API_KEY}" \
  -d '{ "level": "error", "category": "login_fail", "platform": "android",
        "message": "Google idToken audience mismatch",
        "data": { "code": 401, "provider": "google" } }'
# → { "success": true, "data": { "id": "...", "createdAt": "..." } }
```

> **fire-and-forget 로 보내라.** 로그 전송 실패가 게임 흐름(로그인 재시도 등)을 막지 않도록
> `await` 하지 않거나 예외를 흡수한다. 로그가 안 올라가도 게임은 정상 진행돼야 한다.

에러: `level` 이 목록 밖이거나 필드 길이/`data` 크기 초과 → `INVALID_INPUT`(400).
API 키 누락/무효 → `INVALID_API_KEY`(401). 분당 상한 초과 → `RATE_LIMIT_EXCEEDED`(429).

## 2. 로그 보기 (운영자 JWT)

### `GET /projects/{projectId}/logs?level=&category=&platform=&limit=&cursor=`

최신순(createdAt DESC) + `id` 커서 페이지네이션. `limit` 기본 50, 최대 200.

```bash
curl "https://api.bbbase.io/projects/{projectId}/logs?level=error&category=login_fail" \
  -H "Authorization: Bearer {accessToken}"

# CLI
bbbase log:list {projectId} --level error --category login_fail --platform android
```

대시보드 프로젝트 → **로그** 화면에서 레벨·카테고리·플랫폼 필터로 볼 수 있다.

## 3. 보존기간

로그는 기본 **30일** 보관 후 자동 정리된다(무한 증식 방지). 서버 환경변수
`LOG_RETENTION_DAYS` 로 조정.

## 게임 연동 흐름
1. (런타임) 소셜/게스트 로그인 실패 catch 블록에서 `POST .../logs` 로 실패 사유·플랫폼·코드를
   보낸다 — 유저 토큰이 아직 없어도 API 키만으로 된다.
2. fire-and-forget: 로그 전송 자체의 실패는 무시(게임 흐름 우선).
3. (운영, JWT) 대시보드 "로그" 또는 `bbbase log:list` 로 실패 패턴을 모니터링.
