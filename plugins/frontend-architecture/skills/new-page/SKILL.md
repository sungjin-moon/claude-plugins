---
name: new-page
description: Next.js App Router 기반 페이지를 ARCHITECTURE.md 패턴에 맞게 생성합니다. "/new-page", "페이지 만들어줘", "새 페이지 생성", "page.tsx 만들어줘" 요청에 사용하세요. 기존 페이지 수정이 아닌 신규 페이지 생성에만 사용합니다.
argument-hint: '<app> <path>'
---

프로젝트의 계층별 아키텍처 원칙에 따라 새 페이지를 생성해줘.
이 스킬은 특정 프로젝트에 고정돼 있지 않다. **실행 시점에 프로젝트의 컨벤션을 먼저 읽고** 그대로 따른다.

인자: $ARGUMENTS
형식: `<앱이름> <경로>`
예시:

- `/new-page <new-app> v2/orders` → /v2/orders 페이지
- `/new-page <new-app> v2/products/[productId]/options` → 동적 라우트 하위 페이지
- `/new-page <new-app> dashboard` → /dashboard 페이지

> 위 `<new-app>`은 `conventions.json`의 `apps`에서 `tier == new`인 앱 이름으로 치환해 사용한다.

## 0. 컨벤션 로드 (항상 먼저)

1. 프로젝트 루트에서 다음을 읽는다:
   - `ARCHITECTURE.md` — 계층 구조·라우팅·import 규칙·네이밍의 **서술적 원천**. 이 문서가 아래 가정과 충돌하면 **항상 프로젝트 문서가 우선**한다.
   - (폴백) 프로젝트에 `ARCHITECTURE.md`가 없으면 → 플러그인 번들 base `${CLAUDE_PLUGIN_ROOT}/ARCHITECTURE.base.md`를 기본 패턴으로 읽어 따른다. (/frontend-architecture:init 으로 프로젝트 ARCHITECTURE.md를 만들어두면 더 좋다.)
   - `.claude/conventions.json` — `apps`, `packages`, `styling`, `monorepo` 노브
2. `conventions.json`이 없으면 `package.json`(workspaces)·폴더 구조에서 추론하고, 애매하면 **추측하지 말고 사용자에게 물어본다.** `ARCHITECTURE.md`가 없으면 기존 페이지 코드를 직접 관찰해 따른다.
3. 아래에서 `{{...}}`는 컨벤션에서 읽은 값을 의미한다:
   - `{{monorepo.appsDir}}` = 앱 디렉토리 (기본 `apps`)
   - `{{app.tier}}` = 대상 앱의 `tier` (new | legacy)
   - `{{app.framework}}` = 대상 앱 프레임워크 (next | vite ...)
   - `{{app.styledSystemImport}}` = 앱 로컬 스타일 import 경로 (panda 등)
   - `{{packages.api}}` = API 함수 패키지 (예: `@org/api`)

## 사전 확인

1. 대상 앱의 기존 페이지 구조 확인 (`{{monorepo.appsDir}}/<앱이름>/app/` 디렉토리)
2. 같은 깊이의 기존 페이지가 있다면 동일한 패턴 파악
3. 상위 layout.tsx가 있는지 확인
4. **관찰한 패턴이 이 문서의 가정과 다르면, 관찰한 패턴을 따른다.**

> App Router(파일 기반 라우팅) 구조를 가정한다. `{{app.framework}}`가 Next.js가 아니면 해당 프레임워크의 라우팅 규칙을 `ARCHITECTURE.md`/기존 코드에서 확인해 따른다.

## 생성할 파일 구조

```
{{monorepo.appsDir}}/<앱이름>/app/<경로>/
├── page.tsx              # 페이지 컴포넌트
├── components/           # 이 계층 전용 컴포넌트 폴더
│   └── index.ts          # barrel export
└── (필요 시)
    ├── schemas/           # Zod 유효성 검사 스키마
    ├── hooks/             # 이 계층 전용 훅
    └── utils/             # 이 계층 전용 유틸
```

> 정확한 계층 폴더 규칙은 프로젝트 `ARCHITECTURE.md`를 우선한다. 위는 기본 가정.

## 페이지 작성 규칙

### 신규 앱 (`{{app.tier}} == new`)

```typescript
// page.tsx - 서버 컴포넌트 (기본)
import { css } from '{{app.styledSystemImport}}'

export default function <PageName>Page() {
  return (
    // 페이지 내용
  )
}
```

- 기본적으로 서버 컴포넌트로 작성
- 클라이언트 기능이 필요한 부분은 별도 컴포넌트로 분리하여 `components/`에 배치
- 데이터 fetching은 서버 컴포넌트에서 직접, 클라이언트에서는 `conventions.json`의 `api.queryLib`(예: TanStack Query) 사용

> 스타일 import는 `app.styledSystemImport`를 사용한다. Panda가 아닌 스타일 시스템(css-modules, tailwind 등)이면 `app.styling`에 맞춰 작성.

### ⚠️ 생성 범위 원칙 (반드시 준수)

**페이지 생성 시 아직 존재하지 않는 파일을 import하면 안 된다.** 예를 들어 `queries/`, `constants/`, `types/` 폴더가 아직 없다면, 그 안의 모듈을 import하는 코드를 생성하지 말 것.

올바른 접근:
- `page.tsx`는 `./components`에서만 import
- `components/index.ts`는 빈 barrel이거나 직접 구현한 컴포넌트만 export
- 필터/페이지네이션 같은 복잡한 UI가 필요하다면 해당 컴포넌트 파일도 함께 생성 (import만 추가하고 파일은 안 만드는 건 금지)

```typescript
// ✅ 올바른 예
// page.tsx
import { OrderList } from './components'
export default function OrdersPage() { return <OrderList /> }

// components/index.ts
export { OrderList } from './OrderList'  // OrderList.tsx도 함께 생성

// ❌ 잘못된 예
import { useOrderList } from '../queries'  // queries/ 폴더가 없으면 금지
import { ORDER_TABS } from '../constants'  // constants/ 파일이 없으면 금지
```

### 레거시 앱 (`{{app.tier}} == legacy`)

- 기존 앱의 페이지 패턴 유지 (`app.styling` / 기존 코드 참고)
- 신규 패턴을 억지로 끌어오지 않는다 (호환성 우선)

## 동작

1. 컨벤션 로드 + 대상 앱의 기존 페이지 패턴 확인
2. `page.tsx` + `components/` 폴더 생성
3. 상위 layout.tsx 존재 여부에 따라 layout 추가 필요 여부 안내
4. 해당 페이지의 라우트 경로와 구조 설명
5. 사용 예시 (링크, 네비게이션) 제공
