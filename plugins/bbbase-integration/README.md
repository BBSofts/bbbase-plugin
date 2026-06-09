# bbbase-integration 플러그인

게임 프로젝트에서 BBBase BaaS를 REST로 연동할 때, Claude Code가 BBBase의 호출 규약
(인증 헤더, 응답 envelope, compareMode, 에러코드)을 자동으로 알도록 해주는 스킬
플러그인이다. 매번 BBBase 가이드 문서를 첨부할 필요 없이, 게임 레포에서 "점수 서버에
저장", "랭킹 붙여줘", "닉네임 중복 막아줘" 같은 요청만 하면 스킬이 자동으로 뜬다.

## 게임 레포에서 설치

이 마켓플레이스는 **BBBase 레포 루트**(`.claude-plugin/marketplace.json`)에 있다.
게임 프로젝트의 Claude Code에서 다음을 실행한다(BBBase 레포 경로로):

```
/plugin marketplace add E:\DevArchive\BBBase
/plugin install bbbase-integration@bbbase
```

> 마켓플레이스를 GitHub에 올렸다면 `/plugin marketplace add <org>/<repo>` 형태로도 추가 가능.

설치 후 게임 레포에서 BBBase 연동 작업을 시작하면 `bbbase-integration` 스킬이 트리거된다.

## 구조

```
plugins/bbbase-integration/
  .claude-plugin/plugin.json
  skills/bbbase-integration/
    SKILL.md                          # 인증 2종, envelope, 가장 흔한 호출, 작업별 라우팅
    references/
      setup.md                        # 최초 셋업(계정·프로젝트·API키), 토큰 갱신
      records.md                      # 스키마 정의 + 레코드/엔티티 + compareMode
      leaderboard.md                  # 등록형 리더보드 + 랭킹 조회
      unique-constraints.md           # 닉네임/길드명 중복 금지
      reset-and-audit.md              # 주기 리셋잡 + 감사로그
```

## 내용 갱신 (유지보수 최소화 설계)

스킬은 두 가지를 분담한다:

- **`/docs-json` (라이브 OpenAPI)** — 정확한 경로·필드·타입. 서버가 코드에서 자동
  생성하므로 항상 최신. 스킬은 "정확한 모양은 `{BASE_URL}/docs-json` 을 봐라"라고
  위임한다. → **엔드포인트·필드가 추가돼도 스킬을 안 고쳐도 된다.**
- **이 스킬** — 인증 모델, 응답 envelope, compareMode 등 **동작·의미·함정**. OpenAPI에
  없는 행동 규칙이라 손으로 관리한다. 이건 잘 안 바뀐다.

그래서 BBBase 업데이트 시 스킬을 만질 일은 **근본 규약이 바뀔 때(인증 방식, envelope,
새로운 동작 규칙)뿐**이다. 그럴 때만 SKILL.md / references를 고치고 `plugin.json` 의
`version` 을 올린다. 원본 가이드 `docs/BBBASE_*.md` 와 동기화해 둘 것.

> ⚠️ `/docs-json` 위임이 효과를 보려면 배포 서버에서 Swagger가 켜져 있어야 한다
> (기본 활성, `SWAGGER_DISABLED=1` 이면 꺼짐). 현재 prod(`https://api.bbbase.io/docs-json`)·
> dev(`http://178.105.162.85:4001/docs-json`) 모두 **200 정상**. 혹시 404 가 뜨면 스킬의
> 인라인 내용으로 폴백한다(스킬에 그렇게 지시돼 있음).
