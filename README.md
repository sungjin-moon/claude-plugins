# moonsj-claude-plugins

moonsj 개인 업무 패턴을 담은 Claude Code 플러그인 마켓플레이스.
특정 회사·프로젝트 전용이 아니라, 스킬·서브에이전트를 **컨벤션 구동형**으로 패키징해 **어느 프로젝트에서든** 그 프로젝트의 `ARCHITECTURE.md` + `.claude/conventions.json`만 두면 그대로 동작하도록 만든 것.

> 마켓플레이스 식별자는 `moonsj-tools`. 채운 설정 예시(가상의 `acme` 프로젝트)는 [`examples/example.conventions.json`](./examples/example.conventions.json) — 어디까지나 *한 프로젝트의 예시*일 뿐이다.

## 플러그인

| 플러그인 | 내용 |
| --- | --- |
| **frontend-architecture** | `init`(프로젝트 분석→conventions.json 자동 생성), `new-page`, `new-component`, `new-api`, `review`, `migrate`, `ui-build`, `design-review`, `add-icon`, `sync-postman` + 프론트엔드 에이전트들. 프로젝트 컨벤션을 읽어 동작. |
| **dev-workflow** | `jira-fix`, `jira-done`, `build`, `dev` + `jira-issue-fetcher`. Jira·Slack·빌드·개발서버 워크플로우. |
| **dev-utils** | `typecheck`, `add-lib`. 구조 무관 범용 유틸. |

## 설치

```bash
# 1. 마켓플레이스 추가 (GitHub repo 푸시 후)
/plugin marketplace add <github-user>/moonsj-claude-plugins

# 2. 원하는 플러그인 설치
/plugin install frontend-architecture@moonsj-tools
/plugin install dev-workflow@moonsj-tools
/plugin install dev-utils@moonsj-tools
```

설치 후 스킬은 네임스페이스가 붙는다: `/frontend-architecture:new-page`, `/dev-workflow:jira-fix` 등.

## 새 프로젝트에 적용

1. 위 플러그인 설치.
2. 대상 프로젝트에서 **`/frontend-architecture:init` 실행**. init이 두 가지를 만든다:
   - `.claude/conventions.json` — 프로젝트를 분석해 구조 노브(패키지·앱·스타일 등)를 자동 채우고, 모르는 값만 물어본다.
   - `ARCHITECTURE.md` — 플러그인에 번들된 **base 아키텍처**([`plugins/frontend-architecture/ARCHITECTURE.base.md`](./plugins/frontend-architecture/ARCHITECTURE.base.md))를 이 프로젝트에 맞춰 스캐폴딩한다. **이게 "내 개발 패턴"이 새 프로젝트로 이식되는 지점이다.**
3. 끝. 이제 `/frontend-architecture:new-page` 등이 이 프로젝트의 conventions.json + ARCHITECTURE.md(= 내 패턴)대로 동작한다. ARCHITECTURE.md는 출발점이니 프로젝트에 맞게 수정하면 된다.

> 핵심: **패턴 철학**(어떻게 개발하는가)은 플러그인의 `ARCHITECTURE.base.md`에 담겨 함께 이식되고, **프로젝트 바인딩**(패키지명·포트·Jira키 등)은 각 프로젝트의 `conventions.json`에서 주입된다. 그래서 B 프로젝트에 설치하면 "내가 쓰던 방식 그대로" 개발된다.

자세한 컨벤션 구동 메커니즘은 [CONVENTIONS.md](./CONVENTIONS.md), 패턴 철학 전문은 [ARCHITECTURE.base.md](./plugins/frontend-architecture/ARCHITECTURE.base.md) 참고.

## 로컬 테스트

```bash
claude --plugin-dir ./plugins/frontend-architecture
```
