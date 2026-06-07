---
name: init
description: 현재 프로젝트를 분석해 .claude/conventions.json 초안과 ARCHITECTURE.md(번들된 base 패턴 기반)를 자동 생성합니다. "/init", "컨벤션 설정 만들어줘", "이 프로젝트에 플러그인 적용 셋업", "내 개발 패턴 이 프로젝트에 적용", "conventions.json 생성", "프로젝트 셋업" 요청에 사용하세요. 이 플러그인(frontend-architecture/dev-workflow 등)을 새 프로젝트에 처음 붙일 때 1회 실행합니다.
argument-hint: ''
---

이 마켓플레이스의 컨벤션 구동형 스킬들이 동작하려면 프로젝트에 두 가지가 필요하다: `.claude/conventions.json`(구조 노브)과 `ARCHITECTURE.md`(패턴 철학).
이 스킬은 **현재 프로젝트를 직접 분석해 conventions.json 초안을 만들고, 플러그인에 번들된 base 아키텍처를 이 프로젝트에 맞춰 ARCHITECTURE.md로 스캐폴딩**한다. 그래서 새 프로젝트에 설치하면 "내가 쓰던 개발 패턴 그대로" 동작하게 된다. 자동으로 알 수 없는 값만 사용자에게 묻는다.

> conventions.json 스키마는 마켓플레이스의 `CONVENTIONS.md`, base 패턴은 `${CLAUDE_PLUGIN_ROOT}/ARCHITECTURE.base.md` 참고.

## 0. 이미 있는지 먼저 확인

1. `.claude/conventions.json`이 이미 있으면 → 덮어쓰지 말고, 현재 내용을 보여준 뒤 **갱신할지/누락 필드만 채울지** 사용자에게 묻는다.
2. `ARCHITECTURE.md`가 있으면 읽어서 서술 규칙을 파악하고, 감지 결과와 교차 검증한다.

## 1. 프로젝트 자동 감지 (직접 읽어서 추론)

아래를 **실제 파일에서 관찰**해 채운다. 추측이 아니라 관찰. 못 찾으면 비워두고 2단계에서 질문한다.

### monorepo

- `pnpm-workspace.yaml` / `package.json`의 `workspaces` / `turbo.json` / `nx.json` 존재 여부로:
  - `monorepo.tool`: `turbo` | `nx` | `none`
  - `monorepo.packageManager`: lockfile로 판정 (`pnpm-lock.yaml`→pnpm, `package-lock.json`→npm, `yarn.lock`→yarn)
  - `monorepo.appsDir` / `packagesDir`: workspace 글롭에서 (`apps/*`, `packages/*` 등). 모노레포가 아니면 `appsDir: "."`.

### apps

- `appsDir` 하위 각 디렉토리(또는 단일 앱이면 루트)를 앱으로 보고:
  - `name`: 폴더명(또는 package.json name)
  - `framework`: 의존성으로 (`next`→next, `vite`→vite 등)
  - `port`: `package.json`의 dev 스크립트(`-p 3001`, `--port` 등), `next.config`·`vite.config`에서
  - `styling`: 의존성/설정으로 (`@pandacss/dev`→panda, `tailwindcss`→tailwind, `*.module.css`→css-modules, `@emotion`→emotion)
  - `alias`: `tsconfig.json`의 `compilerOptions.paths` (`@/*`, `@admin/*` 등)
  - `styledSystemImport`: panda면 `panda.config.ts`의 `outdir`로 경로 구성, 아니면 빈 값
  - `tier`: 판단 어려우면 비워두고 질문 (참조 아키텍처 앱 vs 레거시)

### packages

- `packagesDir` 하위 공유 패키지 package.json name을 모아:
  - `scope`: 공통 네임스페이스(`@org`, `@myorg`, `@` 등)
  - `ui` / `lib` / `api`: 이름·역할로 매칭 (ui 컴포넌트 패키지, lib 래퍼, api 함수 패키지). 단일 패키지 프로젝트면 `src` 하위 경로로(`@/components/ui` 등).
  - `uiComponentsImport`: UI 패키지의 실제 export 경로

### api / styling

- `api.client`: 의존성으로 (`ky`/`axios`/없으면 fetch)
- `api.schemaLib`: `zod`/`yup` 등
- `api.queryLib`: `@tanstack/react-query` 등
- `styling.system`: 위 앱 styling 종합
- `styling.semanticTokens`: panda 토큰 설정·ARCHITECTURE.md에 시멘틱 토큰 규칙이 있으면 true

## 2. 자동으로 알 수 없는 값만 질문

다음은 코드에서 알 수 없으므로 **간결하게 사용자에게 묻는다** (모르면 비워두고 넘어가도 됨):

- `migration.referenceApp` / `legacyApps` — 어느 앱이 "신규 참조 패턴"이고 어느 게 레거시인지 (마이그레이션 스킬 쓸 때만 필요)
- `workflow.jira.projectKeys` / `doneStatus` / `aiMarker` — Jira 쓰는 경우만
- `workflow.slack.notifyChannel` / `notifyEpicReporter` — Slack 알림 쓰는 경우만
- `postman.workspaceId` / `collections` — sync-postman 쓰는 경우만
- `icons.*` — 아이콘 빌드 파이프라인 있는 경우만

> 안 쓰는 기능의 섹션은 **굳이 채우지 말고 생략**한다. 모든 필드는 선택이다.

## 3. conventions.json 생성

1. 감지 + 질문 결과를 합쳐 `.claude/conventions.json`을 작성한다 (`.claude/` 없으면 생성).
2. **각 값이 어디서 왔는지**(감지/질문/추정) 한눈에 보고한다. 추정한 값은 "확인 필요"로 표시.

## 4. ARCHITECTURE.md 스캐폴딩 (핵심 — 패턴이 이식되는 지점)

이게 "B 프로젝트에 설치 → 내가 쓰던 방식 그대로 개발"이 성립하는 핵심 단계다. 플러그인에는 프로젝트 중립의 **base 아키텍처**(`${CLAUDE_PLUGIN_ROOT}/ARCHITECTURE.base.md`)가 번들돼 있다. 이 base가 "기본 개발 패턴"이고, 프로젝트는 그걸 자기 ARCHITECTURE.md로 받아 필요한 부분만 덮어쓴다.

1. 프로젝트 루트에 `ARCHITECTURE.md`가 **이미 있으면** → 건드리지 않는다(그쪽이 우선). "이미 ARCHITECTURE.md가 있어 base를 덮지 않았다. base의 패턴 중 반영하고 싶은 게 있으면 알려달라"고만 안내.
2. **없으면** → `${CLAUDE_PLUGIN_ROOT}/ARCHITECTURE.base.md`를 읽어, **이 프로젝트의 conventions.json 값과 감지된 스택에 맞춰 조정**한 뒤 프로젝트 루트에 `ARCHITECTURE.md`로 작성한다:
   - `{{packages.*}}`, `{{monorepo.*}}`, `{{styling.system}}` 등 placeholder를 conventions.json 실제 값으로 치환한다.
   - 프로젝트에 해당 없는 부분(예: 레거시 앱이 없으면 레거시 관련 절, Panda가 아니면 Panda 전제 예시)은 덜어내거나 그 프로젝트 스타일 시스템에 맞게 바꾼다.
   - base의 **패턴 철학과 "왜"는 보존**한다. 단순 치환이 아니라 "이 프로젝트의 아키텍처 가이드"로 읽히게 다듬는다.
3. 사용자에게 알린다: "base 패턴을 이 프로젝트에 맞춰 ARCHITECTURE.md로 생성했다. 이건 출발점이니 자유롭게 수정하면 되고, 이후 모든 스킬이 이 문서를 기준으로 동작한다."

> base를 복사하지 않기로 선택해도(또는 실패해도), 스킬들은 ARCHITECTURE.md가 없으면 플러그인 base를 폴백으로 직접 참조하므로 최소한의 패턴은 보장된다. 하지만 프로젝트에 ARCHITECTURE.md로 두면 팀이 함께 보고 수정할 수 있어 권장된다.

## 5. 마무리 안내

이제 `/frontend-architecture:new-page`, `new-component`, `new-api`, `review` 등을 쓰면 이 프로젝트의 conventions.json + ARCHITECTURE.md를 기준으로, 참조 패턴 그대로 동작한다.

## 주의

- 값을 임의로 **발명하지 마라**. 관찰되면 채우고, 모르면 비우거나 질문한다.
- 단일 프로젝트(모노레포 아님)도 지원: `monorepo.tool: "none"`, `appsDir: "."`, `apps`에 앱 1개.
- 이 스킬은 **읽기 + conventions.json 1개 쓰기**만 한다. 다른 코드를 수정하지 않는다.
