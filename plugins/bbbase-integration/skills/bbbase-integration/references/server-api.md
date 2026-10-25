# 서버 API — 프로젝트 비밀 키

게임 클라가 할 수 없는 일을 **고객의 서버**가 한다: 재화 지급·차감(결제 검증 뒤, 매치 정산 뒤), 서버 전용
컬럼 쓰기, 임의 유저 레코드 읽기, 우편 발송, 제재. 모두 `/projects/{projectId}/server/...` 아래에 있다.

## 키 — 공개 키와 비밀 키는 섞어 쓸 수 없다

| | 공개 키 | 비밀 키 |
|---|---|---|
| 형식 | `bbbase_` + hex | `bbbase_sk_` + hex |
| 어디에 | 게임 클라 빌드(공개 취급) | **고객 서버 환경변수에만** |
| 부를 수 있는 곳 | 게임 클라 API | `/projects/:pid/server/...` 만 |
| 반대쪽에 쓰면 | 서버 API → `401 SECRET_KEY_REQUIRED` | 게임 클라 API → `401 SECRET_KEY_NOT_ALLOWED` |

- 헤더는 둘 다 `X-API-Key`. 서버 API 에는 게임유저 토큰이 필요 없다.
- 서버 API 는 CORS 를 주지 않는다 — 브라우저에서 부를 수 없다. **비밀 키를 게임 빌드·웹 페이지·저장소에 넣지 말 것.**
- 발급(운영자): 대시보드 프로젝트 → API 키 → "비밀 키", 또는 `bbbase apikey:create <projectId> <이름> --secret`.
  원문은 발급 때 한 번만 보인다.
- 요청 제한: IP 단위 제한은 없고 키 단위 분당 600회(서버 설정). FREE 요금제 한도를 넘어도 막지 않는다(쓰기 수는 센다).

## 지급·차감 — `POST /projects/{pid}/server/entities/{entityType}/{entityId}/grant`

```bash
curl -X POST {BASE_URL}/projects/{projectId}/server/entities/user/{userId}/grant \
  -H "X-API-Key: {SECRET_KEY}" -H "Content-Type: application/json" \
  -d '{ "operationId": "order-20261024-0001", "deltas": { "gem": 100 } }'
```

- `deltas` 는 **NUMBER + compareMode=INCREMENT 컬럼만**(아니면 `INVALID_REWARD_COLUMN`, 없는 컬럼은 `UNKNOWN_COLUMN`).
  재화 컬럼은 보통 쓰기 정책을 `SERVER` 로 둔다(클라가 직접 못 바꾸게).
- 음수는 차감. 결과가 0 아래로 내려가면 `409 INSUFFICIENT_BALANCE`(`details.column/balance/delta`)로 **전부** 되돌린다.
- **`operationId` 필수**(1~128자, 프로젝트 전체에서 유일) — 결제 주문 id·원장 id·매치 id 를 그대로 쓰면 된다.
  - 같은 ID + 같은 대상·증감 재전송 → 다시 적용하지 않고 첫 결과(레코드)를 돌려준다. 응답을 못 받았으면 그대로 재시도.
  - 같은 ID + 다른 대상·증감 → `409 OPERATION_ID_CONFLICT`.
  - 실패한 요청은 ID 를 소모하지 않는다. 기록은 30일 보관 — 그보다 오래된 ID 는 새 거래로 적용된다.
- 레코드가 없으면 스키마 기본값으로 만든 뒤 얹는다. 리더보드·리그 코호트도 같은 트랜잭션에서 맞춘다.
- `entityType` 은 user 외(길드 금고 등)도 된다. 감사 로그에 `actor=SERVER_KEY`, `actorId=키 id` 가 남는다.

## 서버 저장·읽기

```bash
# 서버 전용 컬럼 포함 저장 — compareMode 는 그대로 적용(MAX 면 더 클 때만)
curl -X PUT {BASE_URL}/projects/{projectId}/server/entities/user/{userId}/record \
  -H "X-API-Key: {SECRET_KEY}" -H "Content-Type: application/json" \
  -d '{ "data": { "rating": 1250, "league_tier": 2 } }'

# 아무 유저·엔티티 레코드 읽기
curl {BASE_URL}/projects/{projectId}/server/entities/user/{userId}/record -H "X-API-Key: {SECRET_KEY}"
```

- 저장에는 거래 ID 가 없다. 절대값 저장은 그대로 재시도해도 안전하다 — **재시도하면 안 되는 증감은 grant 로.**

## 우편·제재

| 메서드·경로 | 바디 | 같은 기능의 운영자 API |
|---|---|---|
| `POST /server/mails` | `{audience, recipientId?, title, body, attachments?, expiresAt?}` | `mailbox.md` |
| `POST /server/bans` | `{gameUserId, reason?, expiresAt?}` | 제재 이력의 `bannedBy` 는 `key:<키 id>` |
| `DELETE /server/bans/{banId}` | — | 해제(멱등) |

## 키 교체

1. 새 비밀 키 발급 → 2. 서버 설정을 새 키로 바꿔 배포 → 3. `bbbase apikey:list <projectId>`(또는 대시보드)에서
옛 키의 **마지막 사용**이 멈췄는지 확인(분 단위 갱신) → 4. `bbbase apikey:revoke <projectId> <옛 키 id>`.
폐기는 즉시(캐시 포함) 반영되고 되돌릴 수 없다.
