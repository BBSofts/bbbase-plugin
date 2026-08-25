# Unity SDK (있으면 REST 직접 호출 대신 이걸 쓴다)

BBBase 는 Unity 용 **공식 SDK(`.unitypackage`)** 를 제공한다. 프로젝트에 SDK 가
설치돼 있으면(아래로 감지) `UnityWebRequest` 코드를 손수 짜지 말고 SDK API 를 써라.
SDK 가 인증 헤더·envelope 파싱·404 처리·세션 보관을 전부 대신한다.

> **권위:** SDK 사용법의 최신 원문은 `{BASE_URL}/quickstart/unity` 다. 정확한 메서드
> 시그니처는 프로젝트의 `Assets/BBBase/Runtime/**` 소스(XML 문서주석 포함)가 권위.
> 이 문서는 그 위에 "언제 무엇을 쓰는지"를 더한 것이다.

## SDK 설치 여부 감지

다음 중 하나라도 보이면 SDK 가 설치된 것 → SDK 경로로 진행:

- 폴더 `Assets/BBBase/`(특히 `Assets/BBBase/Runtime/BBBase.cs`)
- C# 코드의 `using BBBaseSdk;` 또는 `BBBase.Init()` 호출
- `Packages/manifest.json` 에 없더라도 위가 있으면 설치된 것(`.unitypackage` 임포트형)

SDK 가 **없고** 고객이 SDK 를 원하면: `.unitypackage` 임포트 → 메뉴 **BBBase ▸ Settings**
에서 `baseUrl`/`projectId`/`apiKey` 입력. 의존성(Newtonsoft Json)은 SDK 가 자동 설치한다.
SDK 를 쓰지 않기로 하면 이 문서를 무시하고 다른 `references/*` 의 REST 방식으로 진행한다.

## 핵심 — 모든 호출은 async/await

```csharp
using UnityEngine;
using BBBaseSdk;

public class GameBackend : MonoBehaviour
{
    async void Start()
    {
        BBBase.Init();                       // 앱 시작 시 1회 (Resources/BBBaseSettings 로드)
        if (!BBBase.IsInitialized) return;   // 설정 누락 시 콘솔에 안내

        try
        {
            await BBBase.Auth.LoginGuestAsync();                 // 게스트(기기 식별자 자동)
            var me = await BBBase.Records.LoadMineAsync();       // 내 레코드(없으면 null)
            await BBBase.Records.SaveMineAsync(new { best_time = 4.35, stars = 120 });
            var top = await BBBase.Leaderboards.GetTopEntriesAsync("LEADERBOARD_ID", 10);
        }
        catch (BBBaseException e)
        {
            // 항상 e.Code 로 분기(BBBaseErrorCodes 상수). 메시지는 사람용.
            Debug.LogError($"BBBase {e.Code}: {e.Message}");
        }
    }
}
```

## API 표면 (네임스페이스 `BBBaseSdk`)

| 호출 | 의미 |
|---|---|
| `BBBase.Init()` | 설정 로드·초기화(1회). `BBBase.IsInitialized` 로 확인 |
| `BBBase.IsLoggedIn` / `BBBase.UserId` | 로그인 상태 / BBBase 가 발급한 내 userId |
| `await BBBase.Auth.LoginGuestAsync(deviceId?)` | 게스트 로그인(생략 시 기기 식별자 자동) |
| `await BBBase.Auth.LoginGoogleAsync(idToken)` | 구글(게임 SDK 가 받은 idToken — 받은 즉시) |
| `await BBBase.Auth.LoginAppsInTossAsync(code)` | 앱인토스 |
| `await BBBase.Auth.RefreshAsync()` | 토큰 회전(자동 교체 저장) |
| `await BBBase.Auth.LogoutAsync()` | 서버 폐기 + 로컬 세션 삭제 |
| `await BBBase.Auth.LinkGoogleAsync(idToken)` | 현재 계정에 구글 연동(userId 불변, 세이브 유지) |
| `await BBBase.Auth.LinkGuestAsync(deviceId?)` / `LinkAppsInTossAsync(code)` | 게스트/앱인토스 신원 추가 |
| `await BBBase.Auth.UnlinkAsync(provider)` | 링크 해제(마지막 수단은 `CANNOT_UNLINK_LAST`) |
| `await BBBase.Auth.GetMeAsync()` | `AccountInfo{ UserId, IsGuest, Providers }` |
| `await BBBase.Records.SaveMineAsync(obj)` | 내 레코드 저장(entityType="user") |
| `await BBBase.Records.LoadMineAsync()` | 내 레코드(JObject, 없으면 null) |
| `await BBBase.Records.LoadMineAsync<T>()` | 내 레코드를 타입 T 로(없으면 default) |
| `await BBBase.Records.SaveAsync(type, id, obj)` | 범용 엔티티(guild/season 등) 저장 |
| `await BBBase.Records.DeleteAsync(type, id)` | 삭제(없어도 성공) |
| `await BBBase.Leaderboards.GetTopEntriesAsync(lbId, n)` | Top-N(`RankEntry[]`) |
| `await BBBase.Leaderboards.GetTopEntriesAsync(lbId, n, 0, groupKey)` | 그룹(길드 등) 내 Top-N — `groupByCol` 리더보드 |
| `await BBBase.Leaderboards.GetRankAsync(lbId, entityId)` | 내 순위(없으면 null) |
| `await BBBase.Leaderboards.GetRankAsync(lbId, entityId, groupKey)` | 그룹 내 내 순위 |
| `await BBBase.Leagues.GetMyStatusAsync(leagueId)` | 내 리그 현황(`LeagueStatus`: Tier/Cohort/Rank/Score/Total/Percentile, 없으면 null) |
| `await BBBase.Leagues.GetMyRanksAsync(leagueId, n)` | 내 티어(또는 방) 내 Top-N(`RankEntry[]`) |
| `await BBBase.Leagues.GetStatusAsync(leagueId, entityId)` | 특정 유저 리그 현황 |
| `await BBBase.Leagues.GetRanksAsync(leagueId, entityId, n)` | 특정 유저의 그룹 내 Top-N |
| `await BBBase.Leagues.AcknowledgeResultAsync(leagueId)` | 승급 연출 본 뒤 결과 확인(seen=true) |

> 리그: 점수는 평소처럼 `Records.SaveMineAsync(new { league_points = ... })` 로 저장(자동 반영). `league_tier`/`league_cohort` 는 서버 관리라 직접 쓰지 말 것. 정의·승강 규칙은 운영자가 미리 등록.
> **승급 연출**: `GetMyStatusAsync().LastResult`(`LeagueResult`, 없으면 null)가 지난 사이클 결과. `LastResult.IsPromotion && !LastResult.Seen` 이면 승급 애니메이션 → `AcknowledgeResultAsync(leagueId)` 로 확인. `PrevRank` 로 순위 변화도 연출.

규칙은 REST 와 **100% 동일**하다 — 헤더 2종, envelope, compareMode 병합(`MIN`/`MAX`/
`INCREMENT`), userId 는 서버 발급, 본인 레코드 소유권. SDK 가 그걸 코드로 감쌌을 뿐이다.

## 에러 처리

실패는 `BBBaseException` 으로 던져지고 `e.Code`(`BBBaseErrorCodes` 상수)로 분기한다:

```csharp
catch (BBBaseException e) when (e.Code == BBBaseErrorCodes.DuplicateValue)
{
    // 닉네임 등 유니크 컬럼 중복 — 다른 값 안내
}
```

주요 코드: `DuplicateValue`(409), `UnknownColumn`, `RecordNotFound`, `RateLimitExceeded`,
`Unauthorized`, `Forbidden`, `UserBanned`(403 제재), `NotLoggedIn`(로그인 전 호출),
`NetworkError`(연결 실패).
`LoadMineAsync`/`GetRankAsync` 는 "없음"을 예외 대신 **null** 로 돌려준다.

서버가 `details` 를 실어보내는 코드는 `e.Details` 로 읽는다 — `UserBanned` 의
`expiresAt`/`reason`, `IdentityAlreadyLinked` 의 `conflictUserId` 등.

## 세션 이벤트 — 재로그인·제재 (SDK 1.12.0+)

예외 코드로 매번 분기하지 말고 이벤트 2개만 구독하면 된다. 둘 다 SDK 가 자동 감지한다.

```csharp
// 액세스·리프레시 토큰이 모두 만료돼 SDK 가 세션을 정리함 → 재로그인 UI
BBBase.SessionExpired += provider => GotoLogin(provider);

// 운영자가 이 계정을 제재함 → 플레이 중단 + 정지 안내
BBBase.Banned += (expiresAt, reason) => {
    Time.timeScale = 0f;
    banPopup.Show(expiresAt, reason);   // expiresAt == null 이면 영구 제재
};
```

`Banned` 는 동시 요청이 403 을 여러 번 받아도 **1회만** 방출된다(정지 팝업 중복 방지).
SDK 는 제재 시 토큰을 지우지 않는다 — `auth/me` 는 제재 중에도 열려 있고, 기간제 제재가
풀리면 재로그인 없이 복구돼야 하기 때문이다. 정지 화면 표시는 게임의 몫이다.

## 자주 막히는 부분

| 증상 | 해결 |
|---|---|
| `CS0246: Newtonsoft ...` | 의존성 자동 설치가 끝나기 전. 기다리거나 Unity 재시작. 안 되면 `com.unity.nuget.newtonsoft-json` 수동 추가 |
| 임포트 직후 Burst `Failed to resolve assembly` | 연쇄 재컴파일 중 일시적 캐시 — Unity 재시작하면 사라짐 |
| `BBBase.IsInitialized == false` | 메뉴 BBBase ▸ Settings 에서 projectId/apiKey 미입력 |
| `BBBaseException` code=`NOT_LOGGED_IN` | 로그인 전에 레코드 호출 — `await BBBase.Auth.Login...` 먼저 |
