# 공유 카운터 (로비 현황판·커뮤니티 목표·길드 기여도)

```
BASE_URL : https://api.bbbase.io
```

> 정확한 경로·필드·타입은 `{BASE_URL}/docs-json`(라이브 OpenAPI)이 권위다. 아래는
> 그 위에 **동작·의미·함정**을 더한 것 — 어긋나면 `/docs-json` 을 믿어라.

## 개념 — 여러 플레이어가 함께 올리는 하나의 숫자

로비 현황판("오늘 전체 시도 12,043회"), 커뮤니티 목표("다 같이 100만 마리 잡기"), 길드 기여도,
난이도별 시도/클리어 수처럼 **전 유저가 공유하는 숫자**를 센다.

**왜 레코드로 하면 안 되나:** 이런 숫자를 `entity_records` 에 직접 쓰면, API 키만 있으면
누구나 임의 값(999999)을 넣거나 음수를 넣거나 레코드를 통째로 지울 수 있다. 카운터는
**클라가 값을 쓰지 못하고 `+delta` 만 요청**하게 한다 — 절대값 쓰기·감소·삭제 경로가 아예
없다(카운터는 올라가기만 한다). 증가폭·유저당 상한·구간·공개여부는 서버가 강제하고,
동시에 수천 명이 올려도 정확히 합산된다(단일 UPSERT).

경계선(헷갈리지 마라):
- **카운터** = 공유 숫자. ↔ **유저 레코드**는 유저 **개인** 데이터(내 최고점수·내 골드).
  카운터는 "누가 얼마나 올렸는지"를 값으로 돌려주지 않는다(유저당 상한 판정에만 쓴다).
- **카운터** ↔ **리더보드/리그**: 순위·정렬이 필요하면 리더보드다. 카운터에는 순위 개념이 없다.
- **카운터** ↔ **외부 분석 도구**: 퍼널 세분화·세그먼트 코호트·임의 쿼리 같은 깊은 행동 분석은
  GA4·GameAnalytics 의 몫이다. 카운터는 **게임이 런타임에 읽어 화면에 띄우고 보상과 연결하는
  숫자**를 담당한다.

## 1. 카운터 등록 (운영자 JWT)

```
POST   /projects/:pid/counters                    정의 생성
GET    /projects/:pid/counters                    목록
GET    /projects/:pid/counters/:name              정의 + 현재 구간 값(OPERATOR 카운터도 보임)
GET    /projects/:pid/counters/:name/history?limit=30   구간별 이력(최신순)
PATCH  /projects/:pid/counters/:name              정책 수정
POST   /projects/:pid/counters/:name/reset        현재 구간 값·유저 기여량 초기화
DELETE /projects/:pid/counters/:name              삭제(쌓인 값 cascade)
```

| 필드 | 필수 | 기본 | 설명 |
|---|---|---|---|
| `name` | ✅ | — | `^[a-z][a-z0-9_]*$`, 64자 이하. 프로젝트 내 유일 |
| `description` | | null | 200자 이하 |
| `window` | | `ALL_TIME` | `ALL_TIME`/`DAILY`/`WEEKLY`/`MONTHLY` |
| `timezone` | | `Asia/Seoul` | IANA 이름만. 구간 경계 기준 |
| `groups` | | `[]` | 허용 그룹 화이트리스트(최대 100, 각 `^[A-Za-z0-9_-]{1,64}$`). 빈 배열=그룹 미사용 |
| `maxDelta` | | `1` | 1회 요청 최대 증가폭(1~1,000,000) |
| `perUserLimit` | | null | 한 유저가 한 구간에 더할 수 있는 총량. 생략=무제한 |
| `visibility` | | `PUBLIC` | `PUBLIC`=게임 클라도 읽음 / `OPERATOR`=운영자만 |

```bash
curl -X POST "https://api.bbbase.io/projects/{PROJECT_ID}/counters" \
  -H "Content-Type: application/json" -H "Authorization: Bearer {accessToken}" \
  -d '{ "name": "today_attempts", "window": "DAILY", "timezone": "Asia/Seoul",
        "groups": ["easy","normal","hard"], "maxDelta": 1, "perUserLimit": 200 }'
```

- ⚠️ **`window` 와 `timezone` 은 생성 후 불변**(이미 쌓인 구간 키의 의미가 달라지므로). 바꾸려면
  새 이름으로 카운터를 하나 더 만든다. `PATCH` 로 바꿀 수 있는 건 `description`, `groups`,
  `maxDelta`, `perUserLimit`, `visibility` 뿐이다.
- 대시보드 프로젝트 → **카운터** 화면 / CLI `counter:*` 로도 등록·수정·초기화할 수 있다.

### ⚠️ `maxDelta`·`perUserLimit` 은 반드시 설정하라 (어뷰징 방어)

API 키는 게임 클라에 임베드되는 **공개 취급** 값이라 요청 자체는 누구나 흉내 낼 수 있다.
카운터가 막아주는 건 "임의 값 쓰기"까지고, **"+1 을 100만 번 보내기"는 상한으로 막는다.**

- `maxDelta` — 게임이 한 번에 올리는 **정직한 최대치**로 맞춘다(판당 +1 이면 `1`). 넉넉하게
  열어두면 그만큼 한 방에 부풀릴 수 있다.
- `perUserLimit` — 한 유저가 한 구간에 낼 수 있는 **현실적 상한**(예: 하루 200판). 정상 유저는
  닿지 않고 매크로만 닿는다.

## 2. 값 올리기 (API 키 + **게임유저 토큰 필수**)

### `POST /projects/{projectId}/counters/{name}/incr`

게임유저 토큰이 **필수**다 — 유저당 상한 판정과 제재 적용에 신원이 필요하다.

```bash
curl -X POST "https://api.bbbase.io/projects/{PROJECT_ID}/counters/today_attempts/incr" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: {API_KEY}" -H "Authorization: Bearer {게임유저 accessToken}" \
  -d '{ "delta": 1, "group": "easy" }'
# → { "success": true, "data": { "name": "today_attempts", "group": "easy",
#       "window": "DAILY", "windowKey": "2026-09-20", "value": 2 } }
```

- `delta` 생략 시 1. **0·음수 불가**(카운터는 올라가기만 한다). `maxDelta` 초과 시
  `INVALID_COUNTER_DELTA`(400).
- 카운터에 `groups` 가 있으면 `group` **필수**, 없으면 **보내면 안 된다**(둘 다 400 `INVALID_INPUT`).
- 응답 `value` 는 **그 그룹의 전체 누적값**(내가 올린 양이 아니다) → 올린 직후 재조회 없이
  화면을 갱신할 수 있다.
- 매 프레임·매 액션마다 올리지 말고 **판 종료 같은 의미 있는 시점에 모아서** 한 번 올려라
  (쓰기 호출로 요금제 사용량에 집계된다).

## 3. 값 읽기 (API 키만 — 로그인 전에도)

### `GET /projects/{projectId}/counters/{name}/value?group=`

`X-API-Key` 만 필요(게임유저 토큰 불필요) → **로그인 화면 이전 로비/타이틀에서도** 현황판을
그릴 수 있다. `visibility=PUBLIC` 인 카운터만 읽힌다.

```bash
curl "https://api.bbbase.io/projects/{PROJECT_ID}/counters/today_attempts/value" \
  -H "X-API-Key: {API_KEY}"
# → { "success": true, "data": {
#       "name": "today_attempts", "window": "DAILY", "windowKey": "2026-09-20",
#       "timezone": "Asia/Seoul",
#       "values": [ { "group": "easy", "value": 2 }, { "group": "normal", "value": 0 },
#                   { "group": "hard", "value": 1 } ],
#       "total": 3 } }

# 그룹 미사용 카운터 → values: [ { "group": null, "value": 25 } ]
```

- `group` 생략 시 **정의된 모든 그룹**이 나온다(아직 값이 없는 그룹은 `0`). `total` 은 합계.
- **읽기에는 5초 캐시**가 있다. 증가 시 즉시 무효화되므로 내가 올린 값은 보통 바로 보이지만,
  **다른 유저가 올린 값은 최대 5초 늦게** 보일 수 있다. 현황판을 초 단위로 폴링하지 마라 —
  화면 진입 시 1회, 필요하면 수 초~수십 초 주기면 충분하다.

## 4. 구간(window)과 이력

값은 **구간별로 따로** 쌓인다. `DAILY` 면 카운터의 `timezone` 자정에 새 구간이 시작되고,
**지난 구간 값은 지워지지 않고 이력으로 남는다**(운영자 `GET .../history`, 기본 400일 보존).
`perUserLimit` 도 구간마다 새로 시작한다.

```jsonc
// GET .../counters/today_attempts/history?limit=5   (운영자 JWT)
{ "success": true, "data": [
    { "windowKey": "2026-09-20", "group": "easy", "value": 2, "updatedAt": "..." },
    { "windowKey": "2026-09-20", "group": "hard", "value": 1, "updatedAt": "..." } ] }
```

## 5. 실전 예제

### (a) 퍼즐 게임 로비 현황판 — 난이도별 시도·클리어·클리어율

1. (운영자, 1회) `today_attempts`, `today_clears` 를 `window: DAILY`,
   `groups: ["easy","normal","hard"]`, `maxDelta: 1`, `perUserLimit: 200` 으로 등록.
2. (게임) 판이 끝날 때 `today_attempts` +1, 클리어했으면 `today_clears` 도 +1 (같은 `group`).
3. (로비) `/value` 를 두 번 읽어 그룹별 시도·클리어·클리어율(`clears/attempts`) 표시.
   API 키만 필요하므로 로그인 전에도 그릴 수 있다.

시도와 클리어의 **그룹 이름을 맞춰두면** 클리어율이 나눗셈 한 번이다.

### (b) 커뮤니티 목표 — "다 같이 100만 마리"

```bash
# 등록: 한 판에서 최대 50마리, 한 유저가 총 500마리까지 기여
-d '{ "name": "community_goal", "window": "ALL_TIME", "maxDelta": 50, "perUserLimit": 500 }'
# 게임: 판이 끝날 때  POST .../community_goal/incr  { "delta": 25 }   (group 없음)
# 로비: GET .../community_goal/value → data.total / 1000000 으로 진행바
```

⚠️ **목표 달성 보상을 클라가 판정해 지급하지 마라.** 값은 누구나 읽을 수 있으므로
"100만 도달 → 클라가 보상 지급"은 위조된다. 운영자가 달성을 확인한 뒤 **우편함 전체발송**으로
주는 것이 안전하다(`references/mailbox.md`).

## SDK 사용 (설치돼 있으면 REST 대신)

⚠️ 인자 순서는 **`(카운터 이름, group, delta)`** 다 — delta 가 두 번째가 아니다. 그룹 없는
카운터에 delta 만 줄 때 Unity 는 `delta:` 명명인자, Godot 은 group 자리에 `""` 를 넣는다.

- **Unity** (SDK **1.13.0 이상**) — 실패는 **`BBBaseException`(`e.Code`)으로 던져진다**(반환값에
  `Ok`/`ErrorCode` 가 없다):
  ```csharp
  await BBBase.Counters.IncrementAsync("today_attempts", "easy");          // +1
  await BBBase.Counters.IncrementAsync("community_goal", delta: 25);       // 그룹 없음

  CounterSnapshot s = await BBBase.Counters.GetValueAsync("today_attempts");
  long easy = s.ValueOf("easy");   // 없으면 0. 전체 합은 s.Total

  try { await BBBase.Counters.IncrementAsync("community_goal", delta: n); }
  catch (BBBaseException e) when (e.Code == BBBaseErrorCodes.CounterLimitExceeded) {
      ShowToast("오늘 기여 한도에 도달했습니다");   // 재시도 금지
  }
  ```
  타입: `CounterValue { Name, Group, Window, WindowKey, Value(long) }`,
  `CounterSnapshot { Name, Window, WindowKey, Timezone, Values(CounterGroupValue[]), Total(long), ValueOf(group) }`.
  상수: `BBBaseErrorCodes.CounterNotFound` / `CounterLimitExceeded` / `InvalidCounterDelta`.
- **Godot** (SDK **1.12.0 이상**) — `BBBaseResult` 반환(`res.ok`, `res.data`(Dictionary), `res.error_code`).
  Unity 의 `ValueOf` 같은 메서드는 **없다** — `res.data.get("values", [])` 배열을 직접 돈다:
  ```gdscript
  await BBBase.counters.increment("today_attempts", "easy")        # +1
  await BBBase.counters.increment("community_goal", "", 25)        # 그룹 없음

  var res := await BBBase.counters.get_value("today_attempts")
  if res.ok:
      for v in res.data.get("values", []):
          print(v.get("group"), " = ", int(v.get("value", 0)))
      print("합계 ", res.data.get("total", 0))
  elif res.error_code == BBBaseErrorCodes.COUNTER_LIMIT_EXCEEDED:
      pass   # 재시도 금지 — UI 안내만

  # 값 하나만 필요하면 편의 함수
  var hard: int = await BBBase.counters.get_total_or("today_attempts", "hard")
  ```
  상수: `BBBaseErrorCodes.COUNTER_NOT_FOUND` / `COUNTER_LIMIT_EXCEEDED` / `INVALID_COUNTER_DELTA`.

그 아래 버전 SDK 에는 카운터 API 가 없다 — SDK 를 올리거나 REST 로 직접 호출하라.

## 에러코드

| code | 의미 | 대처 |
|---|---|---|
| `COUNTER_NOT_FOUND` (404) | 그 이름의 카운터 없음 | 운영자가 먼저 등록(이름 오타 확인) |
| `COUNTER_DUPLICATE` (409) | 같은 이름으로 생성(운영자) | 기존 정의를 `PATCH` 로 수정 |
| `INVALID_COUNTER_DELTA` (400) | `delta` 가 `maxDelta` 초과 | 증가분을 정책에 맞추거나 정책 조정(운영자) |
| `COUNTER_LIMIT_EXCEEDED` (429) | 유저당 구간 상한 초과 | **재시도 금지**(그 구간 내내 실패). `details.perUserLimit`·`details.windowKey` 로 UI 안내만 하고 게임 진행은 계속 |
| `INVALID_INPUT` (400) | 그룹 누락/화이트리스트 밖/그룹 없는데 전송, `delta` 0·음수 | 카운터 정의의 `groups` 확인 |
| `FORBIDDEN` (403) | `visibility=OPERATOR` 카운터를 게임 API 키로 읽음 | 운영자 토큰으로만 조회 |
| `UNAUTHORIZED` (401) | 증가에 게임유저 토큰 누락 | 로그인 후 `Authorization` 헤더 첨부 |
| `USER_BANNED` (403) | 제재된 계정의 증가 시도 | 재시도 금지, 정지 안내(`references/game-auth.md`) |
