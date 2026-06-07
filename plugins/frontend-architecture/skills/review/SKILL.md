---
name: review
description: ARCHITECTURE.md 기준으로 코드의 아키텍처 컨벤션 준수 여부를 리뷰합니다. "/review", "리뷰해줘", "코드 리뷰", "컨벤션 체크", "아키텍처 검토" 요청에 사용하세요. 특정 경로나 인자 없이 "/review"만 입력하면 git diff 기준으로 변경된 파일만 리뷰합니다.
argument-hint: '[path]'
---

프로젝트의 아키텍처 컨벤션을 기준으로 코드 리뷰를 해줘.
이 스킬은 특정 프로젝트에 고정돼 있지 않다. **실행 시점에 프로젝트의 컨벤션을 먼저 읽고** 그 규칙을 기준으로 검사한다.

대상: $ARGUMENTS
예시:

- `/review apps/admin/app/v2/products` → 특정 디렉토리 리뷰
- `/review apps/web` → 앱 전체 리뷰
- `/review` → 현재 git diff 기준 변경된 파일만 리뷰

## 0. 컨벤션 로드 (항상 먼저)

1. 프로젝트 루트에서 다음을 읽는다:
   - `ARCHITECTURE.md` — **리뷰 기준의 1차 원천.** 계층 구조·import 규칙·API 패턴·네이밍·디자인 토큰 규칙이 모두 여기에 서술돼 있다. 리뷰는 이 문서의 규칙을 기준으로 수행한다.
   - (폴백) 프로젝트에 `ARCHITECTURE.md`가 없으면 → 플러그인 번들 base `${CLAUDE_PLUGIN_ROOT}/ARCHITECTURE.base.md`를 기본 패턴으로 읽어 따른다. (/frontend-architecture:init 으로 프로젝트 ARCHITECTURE.md를 만들어두면 더 좋다.)
   - `CLAUDE.md`(있으면) — 프로젝트 전역 규칙 보강.
   - `.claude/conventions.json` — `apps`(tier), `packages`, `styling`, `migration`, `monorepo` 등 구조화된 노브.
2. `conventions.json`이 없으면 `package.json`(workspaces)·폴더 구조에서 추론하고, 애매하면 **추측하지 말고 사용자에게 물어본다.**
3. `ARCHITECTURE.md`가 없으면 → 기존 코드 패턴을 직접 관찰해 기준을 잡는다.
4. 아래에서 `{{...}}`는 컨벤션에서 읽은 값을 의미한다:
   - `{{packages.scope}}` = 공유 패키지 네임스페이스 (예: `@org`)
   - `{{packages.ui}}` / `{{packages.uiComponentsImport}}` = 공용 UI 패키지·컴포넌트 import 경로
   - `{{packages.lib}}` / `{{packages.api}}` = 외부 라이브러리 래퍼 / API 함수 패키지
   - `{{app.styledSystemImport}}` = 앱 로컬 스타일 import 경로 (panda 등)
   - `{{migration.legacyApps}}` / `{{migration.referenceApp}}` = 레거시 앱 목록 / 신규 참조 앱
   - `{{appsDir}}` = `monorepo.appsDir` (기본 `apps`)

> **아래 체크 항목은 일반적인 가정이다. 항목과 ARCHITECTURE.md의 서술이 충돌하면 항상 ARCHITECTURE.md를 따른다.** 공용 컴포넌트 목록·디자인 토큰 규칙 등 프로젝트 고유 규칙은 ARCHITECTURE.md + conventions.json에서 읽어 적용한다.

## 인자가 없는 경우

`git diff --name-only`로 변경된 파일 목록을 확인하고, 해당 파일들만 리뷰.

## 체크 항목

### 1. Import 규칙 위반

- [ ] 신규(tier=new) 앱에서 레거시 패키지 직접 import 여부 — `conventions.json`의 `migration.importMap` 키(예: `utils`, `ui`, `types`)에 해당하는 import가 신규 패턴으로 교체됐는지
- [ ] 외부 라이브러리 직접 import 여부 (`{{packages.lib}}`를 통하지 않고 직접 import)
- [ ] 스타일 시스템을 공용 패키지가 아닌 앱 로컬에서 import하는지 — `{{app.styledSystemImport}}`(예: `@<앱alias>/ui/styled-system/css`) 기준. (`styling.system`이 panda인 경우)

### 2. 계층별 아키텍처 위반

- [ ] 특정 계층 전용 모듈이 잘못된 위치에 배치되었는지
- [ ] 여러 계층에서 사용하는 모듈이 특정 계층에만 있는지
- [ ] 앱 전역 모듈이 `ui/`, `hooks/` 등에 올바르게 배치되었는지

> 정확한 계층 폴더 규칙은 ARCHITECTURE.md를 우선한다.

### 3. API 패턴

- [ ] API 함수를 `{{packages.api}}`에서 import하는지 (앱 로컬 `api/`에 함수 정의 금지)
- [ ] 클라이언트 컴포넌트에서 API 직접 호출 여부 (`api.queryLib`(예: TanStack Query) 필수)
- [ ] API 함수 호출 시 클라이언트 인스턴스(`clientApi`)를 첫 번째 인자로 전달하는지 (`api.clientArgFirst`인 경우)
- [ ] 앱 로컬 `api/`에 클라이언트 인스턴스 설정만 있는지 (API 함수/타입 정의 금지)
- [ ] 요청 타입에 스키마(`api.schemaLib`, 예: Zod)가 정의되어 있는지

### 4. 네이밍 컨벤션

- [ ] 컴포넌트 PascalCase, 훅 useXxx, 스키마 xxxSchema, 타입 xxx.types.ts (ARCHITECTURE.md의 네이밍 규칙 기준)
- [ ] 폴더명 통일 (components, schemas, hooks, types, utils, constants 등)

### 5. 컴포넌트 패턴

- [ ] 불필요한 'use client' 디렉티브
- [ ] 서버 컴포넌트에서 할 수 있는 작업을 클라이언트에서 처리하고 있는지

### 6. 공용 UI 컴포넌트 활용

> 프로젝트 고유 컴포넌트 목록·권장 사용처는 ARCHITECTURE.md(및 `{{packages.uiComponentsImport}}`의 실제 export)에서 읽어 적용한다. 아래는 일반적 예시 — ARCHITECTURE.md에 명시된 규칙으로 대체한다.

- [ ] 라벨 + 입력 필드 조합에 전용 `Field` 류 컴포넌트를 사용하는지 (직접 텍스트 + 입력 나열 대신)
- [ ] 날짜 범위/단일 날짜 선택에 전용 컴포넌트를 사용하는지 (직접 `<input type="date">` 나열 대신)
- [ ] 체크박스 그룹에 전용 그룹 컴포넌트를 사용하는지 (직접 `<div>` + grid CSS 대신)
- [ ] 상태/카테고리 표시에 전용 `Tag` 류 컴포넌트를 사용하는지 (직접 `<span>` + 배경색 대신)

### 7. 코드 품질

- [ ] `console.log`, `console.error`, `console.warn` 등 디버깅 코드 잔존 여부 (파일명과 라인 번호 명시)
- [ ] 선언만 하고 실제로 사용되지 않는 dead code (함수, 변수, 상태, 핸들러)
- [ ] 사용되지 않는 import 여부
- [ ] 하드코딩된 hex 색상값 (`#xxxxxx`) — 시멘틱 토큰 사용 필수 (`styling.semanticTokens`인 경우)
- [ ] 베이스 토큰(예: `Gray-50`, `Pink-50`) 대신 시멘틱 토큰 사용 여부 — ARCHITECTURE.md의 시멘틱 토큰 규칙 섹션(`styling.tokenDocSection`) 참조. 신규 앱: 스타일 시스템 토큰(예: Panda `Bg.Secondary`), 레거시 앱: 해당 앱의 토큰 헬퍼(예: SCSS `token('Bg-Secondary')`)
- [ ] Spacing/Radius 하드코딩 여부 — 토큰이 있으면 시멘틱 토큰 사용
- [ ] mapper 분리: 쿼리의 `select` 내 인라인 변환 로직 → 별도 mapper 함수로 분리 여부
- [ ] View 타입이 query/mutation 파일에 직접 정의되어 있지 않은지 (mappers/ 또는 types/ 배치 원칙)

## 출력 형식

```
## 리뷰 결과

### 위반 사항 (수정 필요)
- [파일경로:라인] 설명 및 수정 방안

### 개선 제안 (선택)
- [파일경로:라인] 설명 및 개선 방안

### 통과
- 문제 없는 항목 요약
```

위반 사항이 있으면 수정할지 물어봐줘.
