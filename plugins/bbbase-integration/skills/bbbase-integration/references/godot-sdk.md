# Godot SDK (있으면 REST 직접 호출 대신 이걸 쓴다)

BBBase 는 Godot(4.1+/GDScript) 용 **공식 SDK(`addons/bbbase`)** 를 제공한다. 프로젝트에
SDK 가 설치돼 있으면(아래로 감지) `HTTPRequest` 코드를 손수 짜지 말고 SDK API 를 써라.
SDK 가 인증 헤더·envelope 파싱·404 처리·세션 보관을 전부 대신한다.

> **권위:** SDK 사용법의 최신 원문은 `{BASE_URL}/quickstart/godot` 다. 정확한 시그니처는
> 프로젝트의 `addons/bbbase/runtime/**` 소스가 권위. 이 문서는 그 위에 "언제 무엇을
> 쓰는지"를 더한 것이다.

## SDK 설치 여부 감지

다음 중 하나라도 보이면 SDK 가 설치된 것 → SDK 경로로 진행:

- 폴더 `addons/bbbase/`(특히 `addons/bbbase/runtime/bbbase.gd`)
- GDScript 코드의 `BBBase.init()` 호출 또는 autoload `BBBase` 사용
- `project.godot` 의 `[autoload]` 에 `BBBase=...` 또는 `[editor_plugins]` 에 `bbbase`

SDK 가 **없고** 고객이 SDK 를 원하면: zip 받아 `addons/bbbase` 를 `res://addons/` 로 복사 →
Project Settings ▸ Plugins 에서 **BBBase** Enable → 메뉴 Project ▸ Tools ▸ **BBBase: 설정
열기/생성** 으로 `res://bbbase_settings.tres` 만들고 `base_url`/`project_id`/`api_key` 입력.
외부 의존성 없음(Godot 내장 기능만). SDK 를 쓰지 않기로 하면 이 문서를 무시하고 다른
`references/*` 의 REST 방식으로 진행한다.

## 핵심 — 모든 호출은 `await`, 반환은 `BBBaseResult`

GDScript 엔 예외(try/catch)가 없다. **모든 SDK 호출은 `BBBaseResult` 를 반환**한다:
`res.ok`(성공 여부), `res.data`(성공 시 데이터, 없으면 null), `res.error_code`/`res.error_message`(실패 시).

```gdscript
extends Node

func _ready() -> void:
    BBBase.init()                          # 앱 시작 시 1회 (res://bbbase_settings.tres 로드)
    if not BBBase.is_initialized():
        return                             # 설정 누락 시 콘솔에 안내

    var login := await BBBase.auth.login_guest()      # 게스트(기기 식별자 자동)
    if not login.ok:
        push_error("%s: %s" % [login.error_code, login.error_message]); return

    var me := await BBBase.records.load_mine()        # 내 레코드(me.data, 없으면 null)
    await BBBase.records.save_mine({ "best_time": 4.35, "stars": 120 })
    var top := await BBBase.leaderboards.get_top_entries("LEADERBOARD_ID", 10)
    for r in top.data:
        print(r.get("rank"), r.get("entityId"), r.get("score"))
```

## API 표면 (autoload `BBBase`)

| 호출 | 의미 |
|---|---|
| `BBBase.init()` | 설정 로드·초기화(1회). `BBBase.is_initialized()` 로 확인 |
| `BBBase.is_logged_in()` / `BBBase.user_id()` | 로그인 상태 / BBBase 가 발급한 내 userId |
| `await BBBase.auth.login_guest(device_id?)` | 게스트 로그인(생략 시 `OS.get_unique_id()`) |
| `await BBBase.auth.login_google(id_token)` | 구글(게임이 받은 idToken — 받은 즉시) |
| `await BBBase.auth.login_apps_in_toss(code)` | 앱인토스 |
| `await BBBase.auth.refresh()` | 토큰 회전(자동 교체 저장) |
| `await BBBase.auth.logout()` | 서버 폐기 + 로컬 세션 삭제 |
| `await BBBase.records.save_mine(dict)` | 내 레코드 저장(entity_type="user") |
| `await BBBase.records.load_mine()` | 내 레코드(`res.data`, 없으면 null) |
| `await BBBase.records.save(type, id, dict)` | 범용 엔티티(guild/season 등) 저장 |
| `await BBBase.records.load(type, id)` | 범용 엔티티 조회(없으면 data=null) |
| `await BBBase.records.delete(type, id)` | 삭제(없어도 성공) |
| `await BBBase.leaderboards.get_top_entries(lb_id, n)` | Top-N(`res.data` = Array) |
| `await BBBase.leaderboards.get_top_entries(lb_id, n, 0, group_key)` | 그룹(길드 등) 내 Top-N — `groupByCol` 리더보드 |
| `await BBBase.leaderboards.get_rank(lb_id, entity_id)` | 내 순위(없으면 data=null) |
| `await BBBase.leaderboards.get_rank(lb_id, entity_id, group_key)` | 그룹 내 내 순위 |

규칙은 REST 와 **100% 동일**하다 — 헤더 2종, envelope, compareMode 병합(`MIN`/`MAX`/
`INCREMENT`), userId 는 서버 발급, 본인 레코드 소유권. SDK 가 그걸 코드로 감쌌을 뿐이다.

## 에러 처리

실패는 예외가 아니라 `res.ok == false` + `res.error_code`(`BBBaseErrorCodes` 상수)로 온다:

```gdscript
var res := await BBBase.records.save_mine({ "nickname": name })
if not res.ok:
    match res.error_code:
        BBBaseErrorCodes.DUPLICATE_VALUE: show_toast("이미 쓰는 닉네임")
        BBBaseErrorCodes.NOT_LOGGED_IN:   goto_login()
        _:                                push_error(res.error_message)
```

주요 코드: `DUPLICATE_VALUE`(409), `UNKNOWN_COLUMN`, `RECORD_NOT_FOUND`, `RATE_LIMIT_EXCEEDED`,
`UNAUTHORIZED`, `FORBIDDEN`, `NOT_LOGGED_IN`(로그인 전 호출), `NETWORK_ERROR`(연결 실패).
`load_mine`/`load`/`get_rank` 는 "없음"을 에러 대신 **`ok=true, data=null`** 로 돌려준다.

## 자주 막히는 부분

| 증상 | 해결 |
|---|---|
| Plugins 에 BBBase 안 보임 | `addons/bbbase` 가 `res://addons/` 바로 아래 있는지 확인(프로젝트 reload) |
| `BBBase.is_initialized() == false` | 메뉴 Project ▸ Tools ▸ BBBase 설정에서 project_id/api_key 미입력 |
| `res.error_code == NOT_LOGGED_IN` | 로그인 전 레코드 호출 — `await BBBase.auth.login_*` 먼저 |
| 메서드명 `load` 충돌 | 내부 호출은 `self.load(...)`. 외부 `BBBase.records.load(...)` 는 정상 |
