# 컨벤션 구동 메커니즘 (Convention-driven)

이 마켓플레이스의 `frontend-architecture` / `dev-workflow` 플러그인 스킬·에이전트는
**특정 프로젝트에 하드코딩되어 있지 않습니다.**
대신 각 스킬은 실행 시점에 **대상 프로젝트가 제공하는 두 가지**를 읽어 그 프로젝트의 컨벤션대로 동작합니다.

1. **`ARCHITECTURE.md`** (프로젝트 루트) — 계층 구조, import 규칙, API 패턴, 네이밍 등 서술형 규칙의 원천.
2. **`.claude/conventions.json`** (프로젝트 루트) — 패키지 prefix, 앱 목록·포트, 스타일 시스템, Jira/Slack 설정 등 구조화된 "노브".

> 스킬은 항상 이 두 파일을 먼저 찾는다.
> - 둘 다 있으면 → 그대로 따른다.
> - `conventions.json`이 없으면 → `package.json`(workspaces), 폴더 구조에서 추론하고, 애매하면 사용자에게 물어본다.
> - `ARCHITECTURE.md`가 없으면 → 기존 코드 패턴을 직접 관찰해 따른다.

이 방식 덕분에 **다른 프로젝트는 자기 `ARCHITECTURE.md` + `conventions.json`만 준비하면** 같은 스킬을 그대로 쓸 수 있습니다.

---

## `.claude/conventions.json` 스키마

모든 필드는 선택(optional)입니다. 있는 만큼만 채우면 됩니다. 없는 값은 스킬이 추론하거나 물어봅니다.

```jsonc
{
  // 모노레포 도구
  "monorepo": {
    "tool": "turbo",          // turbo | nx | none
    "packageManager": "pnpm", // pnpm | npm | yarn
    "appsDir": "apps",        // 앱이 위치한 디렉토리
    "packagesDir": "packages" // 공유 패키지 디렉토리
  },

  // 앱 목록. 스킬이 대상 앱을 식별하고 포트/스타일/배치 규칙을 적용한다.
  "apps": [
    {
      "name": "admin",
      "port": 3004,
      "tier": "new",            // new(참조 아키텍처) | legacy
      "framework": "next",      // next | vite | ...
      "styling": "panda",       // panda | css-modules | tailwind | emotion ...
      "alias": "@admin",        // 앱 로컬 import alias (styled-system 등)
      "styledSystemImport": "@admin/ui/styled-system/css" // 스타일 import 경로 (panda 등)
    }
  ],

  // 공유 패키지 prefix. @org 자리에 들어가는 값.
  "packages": {
    "scope": "@org",          // 공유 패키지 네임스페이스
    "ui": "@org/ui",          // 공용 UI 컴포넌트 패키지
    "uiComponentsImport": "@org/ui/components", // 컴포넌트 import 경로
    "lib": "@org/lib",        // 외부 라이브러리 래퍼 패키지
    "api": "@org/api"         // API 함수 패키지
  },

  // 스타일링 토큰/규칙 — 시멘틱 토큰을 쓰는 경우 등
  "styling": {
    "system": "panda",
    "semanticTokens": true,    // 시멘틱 토큰 강제 여부
    "tokenDocSection": "ARCHITECTURE.md#디자인-토큰" // 토큰 규칙이 적힌 위치(선택)
  },

  // API 패키지 구조 (@org/api 류) — new-api / api-builder가 사용
  "api": {
    "client": "ky",                          // ky | axios | fetch
    "clientArgFirst": true,                   // API 함수 첫 인자로 client 인스턴스 전달
    "schemaLib": "zod",                       // 유효성 검사 라이브러리
    "pathPattern": "packages/api/<domain>/<version>", // 모듈 배치 규칙
    "domains": ["admin", "internal"],
    "queryLib": "tanstack-query"              // 서버 상태 라이브러리
  },

  // 아이콘 패키지 (add-icon)
  "icons": {
    "package": "@org/ui",
    "componentsDir": "packages/ui/src/components/icon",
    "buildCommand": "pnpm --filter @org/ui build:icons"
  },

  // 레거시 → 신규 마이그레이션 매핑 (migrate)
  "migration": {
    "referenceApp": "admin",   // 신규 패턴 참조 기준 앱
    "legacyApps": ["legacy-web", "legacy-mobile"],
    "importMap": {             // 레거시 import → 신규 import
      "utils": "@org/lib",
      "ui": "@org/ui"
    }
  },

  // Postman 동기화 (sync-postman) — 회사 고유값이라 반드시 프로젝트가 채워야 함
  "postman": {
    "workspaceId": "<postman-workspace-id>",
    "collections": {
      "admin": { "id": "<collection-id>", "baseUrlVar": "AdminApiBaseUrl" }
    }
  },

  // 워크플로우 — dev-workflow 플러그인이 사용
  "workflow": {
    "jira": {
      "projectKeys": ["PROJ"],
      "doneStatus": "In Review",          // 완료 시 전환할 상태
      "aiMarker": "🤖 AI 처리됨"       // 완료 댓글 마커(선택)
    },
    "slack": {
      "notifyChannel": "C0EXAMPLE",   // 알림 채널 ID
      "notifyEpicReporter": true       // 상위 Epic 보고자에게 멘션
    }
  },

  // 문서 위치 (기본값: 프로젝트 루트 ARCHITECTURE.md)
  "architectureDoc": "ARCHITECTURE.md"
}
```

## 새 프로젝트에 적용하는 법

1. 마켓플레이스 추가 후 원하는 플러그인 설치 (README 참고).
2. **`/frontend-architecture:init` 실행** — 프로젝트를 분석해 `.claude/conventions.json` 초안을 자동 생성하고 모르는 값만 물어본다. (수동 작성도 가능: 위 스키마에서 해당 프로젝트에 맞는 값만 채운다.)
3. (권장) 대상 프로젝트 루트에 `ARCHITECTURE.md` 작성 (계층·import·API 규칙 서술).
4. 끝. 이제 `/frontend-architecture:new-page`, `/frontend-architecture:review` 등이 그 프로젝트 컨벤션대로 동작한다.

> 채운 예시(가상의 `acme` 프로젝트)는 [`examples/example.conventions.json`](./examples/example.conventions.json) 참고.
