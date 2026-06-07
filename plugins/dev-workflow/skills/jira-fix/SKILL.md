---
name: jira-fix
description: Jira 이슈를 조회해 작업 범위를 파악하고 수정 작업을 시작합니다. "/jira-fix PROJ-XXXX", "PROJ-XXXX 이슈 수정해줘", "이 티켓 봐줘", "버그 처리", "디자인 QA 이슈 작업", "이슈 분석해줘", "PROJ-XXXX 무슨 작업이야" 등 Jira 이슈 키나 URL이 언급된 요청에는 적극적으로 사용하세요. 댓글 등록·상태 변경은 절대 하지 않고, 마무리(댓글 + 완료 상태 전환)는 `/jira-done`이 담당합니다.
argument-hint: '[issue-key-or-url]'
---

Jira 이슈를 조회해 작업 범위를 파악하고, 코드 수정을 진행할 수 있도록 컨텍스트를 준비해줘.
이 스킬은 특정 프로젝트에 고정돼 있지 않다. **실행 시점에 프로젝트의 컨벤션을 먼저 읽고** 그대로 따른다.

대상: $ARGUMENTS

예시:
- `/jira-fix PROJ-6628`
- `/jira-fix https://<workspace>.atlassian.net/browse/PROJ-6628`
- `/jira-fix PROJ-6628 PROJ-6629` (다중 키 — 순차 처리)

## 0. 컨벤션 로드 (항상 먼저)

1. 프로젝트 루트에서 다음을 읽는다:
   - `ARCHITECTURE.md` — 계층 구조·import 규칙·타입체크 명령 등 서술적 원천
   - `.claude/conventions.json` — `workflow.jira`, `monorepo` 노브
2. 아래에서 `{{...}}`는 컨벤션에서 읽은 값을 의미한다:
   - `{{jira.projectKeys}}` = `workflow.jira.projectKeys` (예: `["PROJ"]`)
   - `{{monorepo.tool}}` / `{{monorepo.packageManager}}` = 빌드·타입체크 명령 베이스 (기본 `turbo` / `pnpm`)
3. `conventions.json`이 없으면 `package.json`(workspaces)·폴더 구조에서 추론하고, **이슈 키 형식·workspace 도메인이 애매하면 추측하지 말고 사용자에게 물어본다.**

## 1) 입력 파싱

`$ARGUMENTS`에서 issue key를 추출.

- URL 형태(`https://...atlassian.net/browse/<KEY>`)면 `<KEY>` 부분만 추출
- 공백/콤마로 구분된 다중 키 지원 (각 키별로 한 번씩 아래 흐름 반복)
- 키 형식은 `[A-Z][A-Z0-9_]+-\d+` (예: `PROJ-1234`, `SHOP-42`). `{{jira.projectKeys}}`가 있으면 그 프로젝트 키들을 우선 매칭 기준으로 삼는다.

## 2) Subagent 호출로 조회·분석

각 issue key에 대해, 이 플러그인의 `jira-issue-fetcher` 에이전트를 호출한다:

```
Agent(
  subagent_type: "jira-issue-fetcher",
  description: "<KEY> 조회·분석",
  prompt: "Jira 이슈 <KEY>를 조회해 작업 범위를 요약해줘. 본문, 첨부 이미지, 추정 변경 파일, 리스크를 보고 형식대로 정리."
)
```

Subagent는 본문·이미지 raw 토큰을 자기 컨텍스트에서 소비하고, **요약만** 메인에 돌려준다 → 메인 컨텍스트 절약.

## 3) 요약 표시 + 변경 대상 확인

Subagent 보고를 받은 뒤:

- 이슈 핵심 1줄
- 요구사항 bullet
- 첨부 이미지에서 읽은 디자인 시사점
- **추정 변경 파일 후보** (경로 + 줄 번호 가능하면)
- 리스크 / 영향 범위

사용자에게 표시하고, 변경 대상이 맞는지 확인. 필요하면 추가 grep/read로 현재 코드를 확인.

## 4) 코드 수정

메인 대화에서 직접 진행:
- Edit / Write 도구로 파일 수정
- 타입체크 통과 확인 — `ARCHITECTURE.md` / `conventions.json`에 명시된 명령을 우선 사용. 명시가 없으면 `{{monorepo.packageManager}} exec tsc --noEmit` (예: `pnpm exec tsc --noEmit`)
- 특정 앱만 검사할 때는 `{{monorepo.packageManager}} --filter <앱> exec tsc --noEmit`

## 5) 완료 안내

수정이 끝나면 다음을 안내:

```
<KEY> 수정 완료. 마무리하려면:
/jira-done <KEY>
```

여러 키를 한꺼번에 처리했다면:

```
/jira-done <KEY-A> <KEY-B> <KEY-C>
```

## 하지 말 것

- **댓글 등록·상태 전환 절대 금지** — 이건 `/jira-done`의 책임
- 이슈 본문을 그대로 사용자에게 dump 하지 않기 (요약만)
- subagent 보고 없이 임의로 파일 추정하지 않기
