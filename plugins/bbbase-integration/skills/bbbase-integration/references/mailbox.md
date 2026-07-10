# 우편함 (Mailbox — 발송·수령, 보상 지급)

```
BASE_URL : https://api.bbbase.io
```

> 정확한 경로·필드·타입은 `{BASE_URL}/docs-json`(라이브 OpenAPI)이 권위다. 아래는
> 그 위에 **동작·의미·함정**을 더한 것 — 어긋나면 `/docs-json` 을 믿어라.

## 개념 — 운영자 발송 + 게임클라 수령

- **운영자**(JWT)가 개인(특정 유저) 또는 전체발송으로 메일을 보낸다(보상 첨부 가능).
- **게임 클라**(API 키 + 게임유저 토큰)는 우편함을 조회하고 수령 버튼에서 `claim` **한 번만**
  호출한다. **재화 지급은 서버가 원자적으로** 한다 — 재수령해도 재화가 늘지 않는다(멱등).

> ⚠️ **보상 컬럼은 `NUMBER` + `compareMode=INCREMENT`(scope=user) 스키마만** 가능하다(발송
> 시점 검증, 아니면 `INVALID_REWARD_COLUMN`). 서버가 "더해주려면" 그 컬럼이 덧셈 의미여야
> 하기 때문 — 절대값 덮어쓰기(`NONE`) 컬럼은 지급 직후 게임이 자기 캐시값을 PUT 하는 순간
> 보상이 증발한다. 그래서 그 컬럼은 게임도 평소 절대값이 아니라 **증감분(+획득/−소비)**으로
> 저장해야 한다. 보상 컬럼은 리더보드 집계 컬럼과 분리 권장(안 그러면 우편 보상이 랭킹에 반영됨).

## 1. 우편 발송·관리 (운영자 JWT)

대시보드 프로젝트 → **우편함** 화면에서 클릭으로 발송·회수할 수 있다(보상 드롭다운엔
NUMBER+INCREMENT 컬럼만 노출). 아래 REST/CLI 는 직접 연동하거나 자동화할 때 참고.

### 발송 — `POST /projects/{projectId}/mails`

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| audience | `USER` \| `ALL` | ✅ | 개인(특정 유저) 또는 전체발송 |
| recipientId | string | audience=USER 일 때 | 대상 게임유저 id |
| title / body | string | ✅ | 제목 / 본문 |
| attachments | object | ❌ | 보상 `{ 컬럼명: 수량 }`. NUMBER+INCREMENT 컬럼만 |
| expiresAt | ISO8601 | ❌ | 만료 시각. 생략 시 무기한(만료 우편은 조회 제외 후 정리) |

```bash
# 개인 메일 + 보상
curl -X POST https://api.bbbase.io/projects/{projectId}/mails \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "audience": "USER", "recipientId": "{userId}",
        "title": "문의 답변 보상", "body": "불편을 드려 죄송합니다.",
        "attachments": { "gold": 500, "gems": 10 } }'

# 전체발송(공지) — 발송 시점 이전 가입자만 대상
curl -X POST https://api.bbbase.io/projects/{projectId}/mails \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "audience": "ALL", "title": "정기 점검 안내", "body": "오늘 22시 점검 예정입니다." }'
```

> **전체발송은 발송 시점 이전 가입자만** 받는다(발송 후 가입한 신규 유저 제외). 유저마다
> 복제하지 않고 정의 1건으로 저장돼 유저 수와 무관하게 즉시 발송된다(O(1)).

관리: `GET /projects/{projectId}/mails`(발송 이력), `DELETE .../mails/{id}`(회수 — 미수령분은
더 이상 안 보임, 이미 수령한 보상은 회수 안 됨). CLI: `bbbase mail:send` / `bbbase mail:list`.

## 2. 우편함 조회·수령 (API 키 + 게임유저 토큰)

대상 판정은 토큰의 gameUserId 만 신뢰한다 — 남의 우편을 건드릴 수 없다.

```bash
GET  /projects/{projectId}/mailbox?includeClaimed=false&limit=50   # 내 우편함(미수령)
POST /projects/{projectId}/mailbox/{mailId}/read                    # 읽음(멱등)
POST /projects/{projectId}/mailbox/{mailId}/claim                   # 수령(원자·멱등)
POST /projects/{projectId}/mailbox/claim-all                        # 전체 수령
```

수령 응답: `{ claimed, alreadyClaimed, attachments }`.
- `claimed=true` → 이번에 지급됨.
- `alreadyClaimed=true` → 이미 받은 우편(재화 불변).

서버가 수령표시와 지급을 한 트랜잭션에서 처리해, 동시에 여러 번 눌러도 **정확히 한 번만**
지급된다. 수령 후 최신 잔액은 유저 레코드를 다시 읽어(`GET .../entities/user/{userId}/record`)
반영한다.

> **SDK 가 있으면 REST 대신 SDK.** Godot: `BBBase.mails.get_mailbox/mark_read/claim/claim_all`
> (`references/godot-sdk.md`). Unity: `BBBase.Mails.GetMailboxAsync/MarkReadAsync/ClaimAsync/ClaimAllAsync`
> (`references/unity-sdk.md`). 수령 후 잔액은 `records.load_mine()` / `Records.LoadMineAsync()`.

## 자주 만나는 에러코드
| code | 의미 | 대처 |
|---|---|---|
| `INVALID_REWARD_COLUMN` (400) | 보상 컬럼이 NUMBER+INCREMENT 아님 | 재화 컬럼을 INCREMENT 로 정의(운영자) |
| `UNKNOWN_COLUMN` (400) | 보상 컬럼이 스키마에 없음 | 먼저 컬럼 정의 |
| `MAIL_NOT_FOUND` (404) | 메일 없음/내 대상 아님 | 우편함 목록 재조회 |
| `MAIL_EXPIRED` (410) | 만료된 메일 수령 시도 | "만료됨" 표시, 목록 갱신 |

## 게임 연동 흐름
1. (1회, JWT) 보상용 컬럼(예 `gold`)을 `NUMBER` + `compareMode=INCREMENT`(scope=user)로 정의.
2. (운영, JWT/대시보드) 개인 또는 전체발송으로 우편 발송(보상 첨부).
3. (런타임) 게임 우편함 화면에서 목록 조회 → 수령 버튼에서 `claim` 한 번 → 잔액 재조회.
