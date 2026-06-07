---
name: dev
description: 앱 개발 서버를 실행합니다. "/dev", "개발 서버 실행", "로컬 서버 켜줘", "앱 띄워줘" 등의 요청에 사용하세요. 컨벤션에 등록된 모든 앱(신규·레거시) 지원.
argument-hint: '[app-name]'
---

$ARGUMENTS 앱의 개발 서버를 실행해줘.
이 스킬은 특정 프로젝트에 고정돼 있지 않다. **실행 시점에 프로젝트의 컨벤션을 먼저 읽고** 그대로 따른다.

## 0. 컨벤션 로드 (항상 먼저)

1. 프로젝트 루트에서 다음을 읽는다:
   - `ARCHITECTURE.md` — 개발 서버·스타일 시스템 등 서술 규칙(있으면)
   - `.claude/conventions.json` — `monorepo`, `apps` 노브
2. 아래에서 `{{...}}`는 컨벤션에서 읽은 값을 의미한다:
   - `{{monorepo.tool}}` = `monorepo.tool` (turbo | nx | none, 기본 `turbo`)
   - `{{monorepo.packageManager}}` = `monorepo.packageManager` (pnpm | npm | yarn, 기본 `pnpm`)
   - `{{apps}}` = `apps` 목록. 각 앱의 `name` / `port` / `tier`(new | legacy) / `styling`을 사용
3. `conventions.json`이 없으면 `package.json`(scripts·workspaces)에서 dev 명령을 추론하고, **앱 식별이 애매하면 추측하지 말고 사용자에게 물어본다.**

## 앱 목록

앱 목록·포트는 **하드코딩하지 않는다.** `conventions.json`의 `apps` 배열에서 읽어 `name (포트 {{port}})` 형태로 보여준다. 예시(컨벤션에 따라 달라짐):

- 신규 앱 (`tier == new`, 보통 Panda CSS 등 codegen 필요 → `{{monorepo.tool}}`로 실행 권장)
- 레거시 앱 (`tier == legacy`)

> 아래 명령은 `{{monorepo.tool}} == turbo`, `{{monorepo.packageManager}} == pnpm` 가정의 예시다. 다른 도구면 그에 맞춰 치환한다(예: nx면 `nx serve <app>`).

## 실행 규칙

먼저 `{{apps}}`에서 해당 앱의 `tier`를 확인한다.

### 신규 앱 (`tier == new`)

스타일 codegen(예: Panda CSS)이 필요하므로 의존 패키지까지 함께 dev 모드로 실행:

```bash
{{monorepo.packageManager}} {{monorepo.tool}} run dev --filter=$ARGUMENTS... --parallel
```

> `--filter=<app>...`의 `...`은 의존 패키지(공용 UI 등)도 함께 dev 모드로 실행한다는 의미. 이걸 빼고 단일 앱만 실행하면 codegen이 동작하지 않아 스타일이 적용되지 않을 수 있다.

### 레거시 앱 (`tier == legacy`)

```bash
{{monorepo.packageManager}} --filter $ARGUMENTS dev
```

### 전체 통합 실행

프로젝트에 통합 실행 스크립트(프록시 + 전체 앱 등)가 정의돼 있으면 그 스크립트를 쓴다. `package.json`의 scripts에서 확인하고, 있으면 안내한다(예: `{{monorepo.packageManager}} merge`). 없으면 개별 앱 실행으로 안내.

앱 이름이 지정되지 않으면 `conventions.json`의 `apps`로부터 사용 가능한 앱 목록(이름 + 포트)을 보여주고 선택하게 해줘.
