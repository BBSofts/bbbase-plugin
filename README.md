# BBBase Claude Code 플러그인 마켓플레이스

[BBBase](https://bbbase.io) — 게임 개발자용 BaaS — 연동을 돕는 Claude Code 스킬.

## 설치

Claude Code 에서:

```
/plugin marketplace add BBSofts/bbbase-plugin
/plugin install bbbase-integration@bbbase
```

설치 후 게임 레포에서 "점수 서버에 저장", "랭킹 붙여줘" 같은 요청을 하면
`bbbase-integration` 스킬이 자동으로 떠 BBBase 호출 규약(인증·envelope·compareMode·
에러코드)을 적용합니다.

## 문서
- 개발 가이드: https://bbbase.io/docs
- 연동 규약(라이브): https://api.bbbase.io/llms.txt
