---
name: ui-build
description: Figma 디자인을 분석하여 ARCHITECTURE.md 기반의 완전한 UI 코드를 자동 생성합니다. "/ui-build", Figma URL을 주면서 "코드 만들어줘", "구현해줘", "페이지/컴포넌트 생성해줘" 요청에 사용하세요. 디자인 검증(design-review)이나 아이콘 추출(add-icon)이 아닌, Figma → 코드 변환이 목적일 때 사용합니다.
argument-hint: '<figma-url> <app> [path]'
---

Figma 디자인 링크를 분석하고 프로젝트 아키텍처에 맞는 UI 코드를 자동으로 생성해줘.
이 스킬은 특정 프로젝트에 고정돼 있지 않다. **실행 시점에 프로젝트의 컨벤션을 먼저 읽고** 그대로 따른다.

인자: $ARGUMENTS
형식: `<Figma 링크> <앱이름> [배치경로]`
예시:

- `/ui-build https://figma.com/design/xxx... <app> v2/orders`
- `/ui-build https://figma.com/design/xxx... <app> dashboard`

## 0. 컨벤션 로드 (항상 먼저)

1. 프로젝트 루트에서 다음을 읽는다:
   - `ARCHITECTURE.md` — 계층 구조·import 규칙·API 패턴·디자인 토큰의 **서술적 원천 (우선)**
   - (폴백) 프로젝트에 `ARCHITECTURE.md`가 없으면 → 플러그인 번들 base `${CLAUDE_PLUGIN_ROOT}/ARCHITECTURE.base.md`를 기본 패턴으로 읽어 따른다. (/frontend-architecture:init 으로 프로젝트 ARCHITECTURE.md를 만들어두면 더 좋다.)
   - `.claude/conventions.json` — `apps`, `packages`, `styling`, `icons`, `monorepo` 노브
2. `conventions.json`이 없으면 `package.json`(workspaces)·폴더 구조에서 추론하고, 애매하면 **추측하지 말고 사용자에게 물어본다.**
3. 아래에서 `{{...}}`는 컨벤션에서 읽은 값을 의미한다:
   - `{{monorepo.appsDir}}` = `monorepo.appsDir` (기본 `apps`)
   - `{{monorepo.packagesDir}}` = `monorepo.packagesDir` (기본 `packages`)
   - `{{app}}` = `apps[]`에서 대상 앱 항목 (name, port, tier, framework, styling, alias, styledSystemImport)
   - `{{packages.ui}}` = `packages.ui` (공용 UI 패키지, 예: `@org/ui`)
   - `{{packages.uiComponentsImport}}` = `packages.uiComponentsImport` (컴포넌트 import 경로, 예: `@org/ui/components`)
   - `{{app.styledSystemImport}}` = 대상 앱의 스타일 import 경로 (예: `@<alias>/ui/styled-system/css`)
   - `{{styling.system}}` = 스타일 시스템 (panda | css-modules | tailwind ...)
   - `{{icons.*}}` = 아이콘 패키지·디렉토리·빌드 명령
4. **두 번째 인자(앱 이름)는 `conventions.json`의 `apps[].name`에 존재해야 한다.** 없으면 사용자에게 확인한다.

> 디자인 토큰·시멘틱 토큰 규칙, 계층 배치 규칙은 항상 `ARCHITECTURE.md`(또는 `styling.tokenDocSection`)를 우선 원천으로 삼는다. 아래 설명은 기본 가정일 뿐이다.

## 작업 위임

이 스킬은 `frontend-ui-builder` agent에게 작업을 위임합니다.

```
Task(
  subagent_type="frontend-ui-builder",
  prompt="
    Figma URL: $ARGUMENTS에서 첫 번째 인자
    앱 이름: $ARGUMENTS에서 두 번째 인자 (conventions.json의 apps[].name 중 하나)
    배치 경로: $ARGUMENTS에서 세 번째 인자 (선택적)

    먼저 ARCHITECTURE.md + .claude/conventions.json을 읽고,
    위 Figma 디자인을 분석하여 그 프로젝트의 컨벤션 기반 완전한 UI 코드를 생성하세요.
  "
)
```

## frontend-ui-builder가 하는 일

1. **컨벤션 로드**

   - `ARCHITECTURE.md` + `.claude/conventions.json` 로드
   - 대상 앱의 `tier`(new | legacy), `styling`, `styledSystemImport`, `alias` 파악

2. **Figma 디자인 분석**

   - Figma MCP를 통해 디자인 데이터 가져오기
   - 레이아웃 구조, 컴포넌트 계층, 스타일 속성 분석

3. **기존 컴포넌트 매핑**

   - `{{packages.uiComponentsImport}}`에서 재사용 가능한 컴포넌트 검색
   - 앱 내부 `ui/components/`에서 재사용 가능한 컴포넌트 검색
   - 새로 만들어야 하는 컴포넌트 식별

4. **아이콘/이모지 처리**

   - Figma 디자인에 사용된 아이콘/이모지를 식별
   - 아이콘이 포함된 Figma 노드를 **개별 drill-down 조회**하여 아이콘 이름(`ic_*`)과 형태를 정확히 파악
   - `{{packages.ui}}`에 해당 컴포넌트가 있는지 확인 (`{{icons.componentsDir}}` 및 동일 패키지의 emoji 디렉토리)
   - 있으면 → import하여 사용
   - 없으면 → `/add-icon` 스킬 워크플로우에 따라 추가 (단일 컬러 → icon, 다중 컬러 → emoji 자동 분류)
   - Figma에서 `ic_*` 형태의 아이콘은 **인라인 SVG 직접 작성 금지** — 반드시 `{{packages.ui}}` 아이콘 컴포넌트로 등록 후 사용
   - `ic_*` 형태가 아닌 장식적 SVG(일러스트, 배경 등)는 인라인 SVG 허용

5. **코드 생성**

   - 신규(new) 앱: page.tsx (서버 컴포넌트), components/\*.tsx (필요시 'use client'), components/index.ts (barrel export)
   - 스타일링: `{{styling.system}}`에 맞춰 작성
     - panda인 경우 → `import { css, cx, cva } from '{{app.styledSystemImport}}'`
     - css-modules / tailwind 등인 경우 → 해당 앱의 기존 패턴을 그대로 따른다
   - 시멘틱 토큰: `styling.semanticTokens`가 true면 베이스 토큰/하드코딩 hex 대신 시멘틱 토큰 사용 (규칙은 `ARCHITECTURE.md` 토큰 섹션 우선)
   - 레거시(legacy) 앱: 해당 앱의 기존 스타일/패키지 패턴 유지 (신규 패턴 억지로 끌어오지 않음)

6. **결과 리포트**
   - 생성된 파일 목록
   - 사용된 `{{packages.ui}}` 컴포넌트
   - 새로 생성한 커스텀 컴포넌트
   - 추가된 아이콘 (있는 경우)
   - 다음 단계 안내

## 디자인 검증 (자연어 트리거)

코드 생성 후 디자인 검증은 사용자의 자연어 요청에 따라 동작 방식이 달라집니다.

### 사용자가 "자동 검증 해줘"라고 말한 경우 → 자동 반복 루프

개발 서버가 실행 중이어야 합니다. 코드 생성 완료 후 자동으로 검증-수정 루프를 실행합니다.

1. Figma 스크린샷 촬영 (`get_screenshot`)
2. Playwright MCP로 로컬 페이지 접속 → 해당 UI 상태 재현 → 스크린샷 촬영
3. 두 스크린샷을 비교하여 차이점 식별
4. 차이점이 있으면 코드 수정 후 다시 2~3 반복
5. **최대 3회 반복** 후 종료, 남은 차이점은 리포트로 보고
6. Playwright 브라우저 닫기

### 사용자가 검증을 언급하지 않은 경우 → 기본 동작

코드 생성 + 결과 리포트만 제공합니다. 사용자가 이후 `/design-review`를 직접 실행하여 검증할 수 있습니다.

### Playwright 설정 참고

- 뷰포트: 1440x900
- 로컬 접속 포트는 대상 앱의 `{{app.port}}`(conventions.json `apps[].port`)를 사용한다. 없으면 dev 서버 로그/설정에서 추론.
- 모달/드롭다운 등 인터랙션이 필요하면 `browser_click` 등으로 재현

## 사용 예시

```bash
# 기본: 코드 생성만
/ui-build https://figma.com/design/abc123/Dashboard?node-id=1-2 <app> /marketing/dashboard

# 자연어로 자동 검증 요청
/ui-build https://figma.com/design/xyz789/Orders?node-id=10-20 <app> v2/orders
# → 이후 "자동 검증 해줘" 라고 말하면 자동 반복 루프 실행
```

## 주의사항

- Figma 로그인 필요 (MCP 권한)
- 생성된 파일은 검토 후 수정 가능
- UI 코드만 생성 (API 연동은 별도)
- 자동 검증 시 개발 서버가 실행 중이어야 함 (대상 앱의 dev 명령은 `ARCHITECTURE.md`/`README`/`package.json` 참고)
