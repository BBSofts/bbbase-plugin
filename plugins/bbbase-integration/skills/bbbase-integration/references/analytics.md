# 분석 — 리텐션 · 이탈율 · 이탈구간(퍼널)

BBBase 는 게임의 **리텐션(코호트별 D1/D7/D30)**, **이탈율**, **이탈구간(퍼널)** 을 대시보드에서
보여준다. 무거운 이벤트 로그 없이 **비트마스크**로 집계하므로 게임 클라가 할 일이 거의 없다.

## 무엇이 자동이고, 무엇을 게임이 신호하나

| 지표 | 누가 | 게임 클라 작업 |
|---|---|---|
| **리텐션(D1/D7/D30)** | 서버 자동 | **없음** — 로그인/레코드 저장만 붙어 있으면 됨 |
| **이탈율** | 서버 자동 | **없음** — 마지막 접속 시각으로 파생 |
| **이탈구간(퍼널)** | 게임이 신호 | 단계 도달 시 `BITSET` 컬럼에 비트 저장 |

조회는 모두 **운영자 JWT**(게임 API 키 아님). 보통 대시보드 프로젝트 → **분석** 화면에서 본다.

## 리텐션 · 이탈율 — 자동 (코드 추가 없음)

게임 유저가 로그인하거나 레코드를 저장하면, 서버가 가입일 기준 "N일째 접속"을 자동으로 비트에
기록하고 매일 새벽(UTC 00:00) 코호트별 D1/D7/D30 을 굽는다. **이미 게스트 로그인 + 레코드 저장이
연동돼 있으면 리텐션은 그냥 쌓인다** — 추가로 보낼 이벤트가 없다.

> D7 = 가입 후 **7일째 되는 날 접속**한 비율(7일 연속이 아님 — 중간에 쉬었어도 그날 오면 카운트).

```bash
# 코호트별 D1/D7/D30 (최근 N일) — 운영자 JWT
GET /projects/{PROJECT_ID}/analytics/retention?days=30
# → [{ cohortDate, cohortSize, d1, d7, d30 }, ...]   D7% = d7 / cohortSize

# 이탈율 — 운영자 JWT
GET /projects/{PROJECT_ID}/analytics/churn?inactiveDays=7
# → { inactiveDays, total, churned, active, churnRate }

# 수동 집계(테스트용 — 평소엔 매일 자동)
POST /projects/{PROJECT_ID}/analytics/retention/run
```

## 이탈구간(퍼널) — 게임이 단계 비트를 저장

"어느 단계에서 떠났나"는 게임만 안다. 도달 단계를 `BITSET` 컬럼에 비트로 모아두면, 서버가 단계별
도달자 수를 집계한다(=퍼널). 한 번 켜진 비트는 OR 로 유지된다.

### 1. BITSET 컬럼 정의 (운영자 — 한 번만)

`compareMode` 셋업이라 운영자 권한이 필요하다(SKILL.md 4.5 — 누가 셋업할지 먼저 물어라).

```bash
curl -X POST {BASE_URL}/projects/{PROJECT_ID}/schemas \
  -H "Content-Type: application/json" -H "Authorization: Bearer {OPERATOR_TOKEN}" \
  -d '{ "columnName": "funnel", "type": "STRING",
        "compareMode": "BITSET", "defaultValue": "0", "scope": "user" }'
# CLI: bbbase schema:create {PROJECT_ID} funnel --type STRING --compare BITSET --default "0"
```

타입은 반드시 **STRING**(큰 비트도 안전하게 10진 문자열로 저장), 기본값 `"0"`.

### 2. 단계 도달 시 비트 저장 (게임 — 평소 레코드 저장과 동일)

단계 N(0~61) 도달 시 `1 << N` 의 **10진 문자열**을 저장한다. 서버가 기존 값과 OR 하므로 동시
호출에도 비트가 유실되지 않는다. 클라가 직접 OR 하지 말 것(읽고-수정-쓰기는 race 위험) — 도달
비트만 보내면 서버가 합친다.

```bash
# 6단계 도달 → 비트6 = 2^6 = "64"
curl -X PUT {BASE_URL}/projects/{PROJECT_ID}/entities/user/{userId}/record \
  -H "Content-Type: application/json" -H "X-API-Key: {API_KEY}" \
  -H "Authorization: Bearer {accessToken}" \
  -d '{ "data": { "funnel": "64" } }'
```

```csharp
// Unity SDK — 튜토리얼/온보딩 단계 도달마다
async Task ReachStage(int stage)            // stage: 0~61
{
    string bit = (1L << stage).ToString();  // 6단계 → "64"
    await BBBase.Records.SaveMineAsync(new { funnel = bit });
}
```

```gdscript
# Godot SDK
func reach_stage(stage: int) -> void:       # stage: 0~61
    await BBBase.records.save_mine({ "funnel": str(1 << stage) })   # 6단계 → "64"
```

**단계 매핑은 고정하라** — 비트0=첫 실행, 1=튜토리얼 시작, 2=1스테이지 클리어… 처럼 일관돼야
분석이 의미를 가진다. 선형 단계(1→2→3)일 때 가장 깔끔하다.

### 3. 단계별 도달자 집계 (운영자)

```bash
GET /projects/{PROJECT_ID}/analytics/funnel?column=funnel
# → [{ stage, reached }, ...]   stage=비트 인덱스=단계 번호. 도달자 0인 단계는 생략.
```

도달자 수가 단계마다 줄어드는 모양이 퍼널이다 — 급격히 꺾이는 구간이 주요 이탈 지점.

## 한계

- 비트마스크는 "여부"만 담는다 → **유입 소스별·국가별 리텐션 비교** 같은 세그먼트 분석은 불가.
- 리텐션 윈도우는 가입 후 약 60일까지.
- 이탈구간은 도달 단계의 **집합**이지 순서가 아니다.

대부분의 운영 지표(코호트 곡선 + 주요 이탈 단계)는 이 방식으로 충분하다. 세그먼트 심층 분석이
필요해지면 그때 이벤트 로그 기반으로 확장한다.
