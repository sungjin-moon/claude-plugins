# 프론트엔드 아키텍처 베이스 (Base Architecture)

> 이것은 새 프로젝트가 상속하는 **base 아키텍처(기본 패턴)**입니다.
> 프로젝트 자체에 `ARCHITECTURE.md`가 있으면 **그쪽이 우선**이고, 이 base는 출발점/폴백으로만 사용합니다.
> 문서 안의 `{{...}}` 표기는 프로젝트 루트의 `.claude/conventions.json` 값으로 해석합니다.
> (예: `{{packages.scope}}` → `@org`, `{{monorepo.appsDir}}` → `apps`)
> 해당 값이 conventions.json에 없으면 스킬이 코드/`package.json`에서 추론하고, 애매하면 사용자에게 물어봅니다.

이 문서는 모노레포 기반 프론트엔드 개발에서 **"모듈을 어디에 둘 것인가, 어떻게 이름 지을 것인가, API/폼/스타일을 어떤 패턴으로 작성할 것인가"**에 대한 일관된 의사결정 규칙을 담습니다. 각 규칙에는 **"왜 그렇게 하는가"**를 함께 적어, 새 상황에서도 같은 원리로 판단할 수 있게 합니다.

---

## 목차

1. [핵심 원칙](#1-핵심-원칙)
2. [모노레포 구조](#2-모노레포-구조)
3. [앱 분류: 신규(new) vs 레거시(legacy)](#3-앱-분류-신규new-vs-레거시legacy)
4. [패키지 분류: 공용 패키지 vs 레거시](#4-패키지-분류-공용-패키지-vs-레거시)
5. [계층별 아키텍처 원칙](#5-계층별-아키텍처-원칙)
6. [폴더 네이밍 컨벤션](#6-폴더-네이밍-컨벤션)
7. [컴포넌트 재사용 가이드](#7-컴포넌트-재사용-가이드)
8. [API 설계 가이드](#8-api-설계-가이드)
9. [폼 검증 패턴 (Zod + RHF)](#9-폼-검증-패턴-zod--rhf)
10. [모듈 배치 의사결정 가이드](#10-모듈-배치-의사결정-가이드)
11. [신규 앱 개발 시 주의사항](#11-신규-앱-개발-시-주의사항)
12. [요약 체크리스트](#12-요약-체크리스트)

---

## 1. 핵심 원칙

모노레포는 여러 프론트엔드 앱과 공유 패키지로 구성됩니다. 모든 코드 배치/네이밍/패턴 결정은 다음 네 가지 원칙에서 파생됩니다.

- **관심사 분리**: 모듈은 그 목적과 사용 범위에 따라 적재적소에 배치한다. "이 모듈을 누가 쓰는가"가 위치를 결정한다.
- **계층별 아키텍처**: URL pathname을 기준으로 계층(Layer)을 나누고, 각 계층에서 필요한 모듈을 그 계층 안에서 관리한다. 사용 범위와 폴더 경로를 일치시킨다.
- **일관된 패턴**: 동일한 폴더 네이밍과 구조를 유지해 코드 탐색 용이성을 확보한다. 어느 앱/계층을 열어도 같은 자리에 같은 종류의 파일이 있어야 한다.
- **점진적 마이그레이션**: 레거시 앱은 유지하면서, 신규 앱은 새로운 패턴으로 개발한다. 신규/레거시를 명확히 구분하고 서로의 규칙을 섞지 않는다.

> 애매한 결정(모듈 배치 위치, 공용 패키지 추가 여부, 신규/레거시 패턴 선택 등)은 추측하지 말고 사용자에게 확인한다.

---

## 2. 모노레포 구조

```
<repo-root>/
├── {{monorepo.appsDir}}/            # 애플리케이션들 (기본값: apps)
│   ├── <new-app-a>/                 # 신규 앱 (tier=new, 참조 아키텍처)
│   ├── <new-app-b>/                 # 신규 앱 (tier=new, 개발 중)
│   ├── <legacy-app-a>/              # 레거시 앱 (tier=legacy)
│   └── <legacy-app-b>/              # 레거시 앱 (tier=legacy)
│
├── {{monorepo.packagesDir}}/        # 공유 패키지들 (기본값: packages)
│   ├── {{packages.scope}}/          # 신규 공용 패키지 스코프
│   │   ├── api/                     # API 엔드포인트 함수 + 팩토리   → {{packages.api}}
│   │   ├── lib/                     # 외부 라이브러리 re-export      → {{packages.lib}}
│   │   ├── ui/                      # UI 컴포넌트 + 스타일 시스템    → {{packages.ui}}
│   │   ├── constant/                # 상수 정의
│   │   └── types/                   # TypeScript 타입 정의
│   │
│   └── <legacy-packages>/           # 레거시 공유 패키지 (config, tsconfig, rest, utils 등)
│
└── <monorepo-config>                # {{monorepo.tool}} 설정 (예: turbo.json)
```

> `{{monorepo.tool}}`(예: turbo/nx)와 `{{monorepo.packageManager}}`(예: pnpm/npm/yarn)는 프로젝트별로 다르므로 conventions.json에서 해석합니다. 이 문서의 명령 예시는 `{{monorepo.packageManager}}` / `{{monorepo.tool}}`로 표기합니다.

---

## 3. 앱 분류: 신규(new) vs 레거시(legacy)

각 앱은 conventions.json의 `apps[].tier` 값으로 **신규(new)** 또는 **레거시(legacy)**로 구분됩니다. 둘은 적용 규칙이 다르므로 작업 전 대상 앱의 tier를 먼저 확인합니다.

### 신규 앱 (tier = new)

- App Router 등 최신 프레임워크 기능 사용
- 프로젝트가 채택한 스타일 시스템(`{{styling.system}}`) 기반 스타일링
- **`{{packages.scope}}/*` 공용 패키지만 사용** (레거시 패키지 직접 import 금지)
- 계층별 아키텍처를 엄격히 적용
- 신규 앱 중 하나를 **참조 아키텍처(`{{migration.referenceApp}}`)**로 삼아, 같은 기능을 새로 만들 때 먼저 참조 앱의 구현을 확인하고 동일 패턴을 따른다.

### 레거시 앱 (tier = legacy)

- 기존 패턴을 유지한다.
- 신규 기능 개발 시 신규 앱 패턴을 참고하되, 기존 구조와의 호환성을 우선 고려한다.
- 레거시 앱에 신규 규칙(`{{packages.scope}}/*` 강제 등)을 무리하게 적용하지 않는다.

> **왜 구분하는가**: 점진적 마이그레이션 전략에서 두 세계가 공존한다. tier를 명시적으로 나눠야 "이 앱에서는 어떤 규칙이 맞는지"를 매번 헷갈리지 않고 판단할 수 있다.

---

## 4. 패키지 분류: 공용 패키지 vs 레거시

### 공용 패키지 (`{{packages.scope}}` 스코프, 신규)

> **중요**: 신규(tier=new) 앱은 반드시 `{{packages.scope}}/*` 공용 패키지만 사용해야 합니다.

| 패키지                  | 용도                 | 주요 내용                                                          |
| ----------------------- | -------------------- | ----------------------------------------------------------------- |
| **{{packages.api}}**    | API 엔드포인트 통합  | 백엔드 API 호출 함수, 요청/응답 타입, 클라이언트 팩토리           |
| **{{packages.lib}}**    | 외부 라이브러리 통합 | 외부 라이브러리(날짜, 폼, 스키마, 서버상태, HTTP 등)의 re-export   |
| **{{packages.ui}}**     | UI 컴포넌트 시스템   | 디자인 토큰, 스타일, 공용 컴포넌트, 공용 훅                        |
| `{{packages.scope}}/constant` | 상수 정의      | 앱 전역에서 사용하는 상수들                                        |
| `{{packages.scope}}/types`    | 타입 정의      | 공통 TypeScript 타입들                                            |

```typescript
// O 올바른 사용 (신규 앱)
import { useForm } from '{{packages.lib}}/react-hook-form'
import { Button } from '{{packages.uiComponentsImport}}'
import { getProductList } from '{{packages.api}}/<domain>'
import { API_BASE_URL } from '{{packages.scope}}/constant'

// X 잘못된 사용 (신규 앱에서 레거시 패키지 직접 import)
import { something } from 'utils' // 사용 금지
import { Component } from 'ui' // 사용 금지
```

### 레거시 패키지

레거시 앱 전용이거나 deprecated 된 패키지(예: 구 `utils`, `ui`, `types`, `constant` 등)는 신규 앱에서 사용하지 않습니다. 신규 패키지로 대체된 항목은 신규 패키지를 사용합니다.

> **왜 그렇게 하는가**: 공용 패키지를 단일 진입점으로 강제하면, 외부 라이브러리 버전/래핑/디자인 시스템을 한곳에서 통제할 수 있다. 앱마다 직접 import하면 버전 드리프트와 중복 구현이 생긴다.

---

## 5. 계층별 아키텍처 원칙

### 핵심 개념

URL pathname을 기준으로 **계층(Layer)**을 나누고, 각 계층에서 필요한 모듈을 그 계층 위치에 배치합니다. **"사용 범위 = 폴더 경로"**가 되도록 만드는 것이 핵심입니다.

```
app/
├── layout.tsx           # 전역 레이아웃
├── page.tsx             # 루트 페이지
│
└── v2/                  # /v2 계층
    ├── layout.tsx
    ├── page.tsx
    │
    ├── products/        # /v2/products 계층
    │   ├── page.tsx
    │   ├── components/   # 이 계층에서만 사용하는 컴포넌트
    │   ├── schemas/      # 이 계층에서만 사용하는 스키마
    │   ├── hooks/        # 이 계층에서만 사용하는 훅
    │   │
    │   ├── [productId]/  # /v2/products/:productId 계층
    │   │   ├── page.tsx
    │   │   ├── components/
    │   │   └── schemas/
    │   │
    │   └── new/          # /v2/products/new 계층
    │       ├── page.tsx
    │       └── components/
    │
    └── orders/          # /v2/orders 계층
        ├── page.tsx
        ├── components/
        └── schemas/
```

> 위의 `v2/`, `products/`, `orders/` 등은 **중립 예시**입니다. 실제 계층명은 프로젝트의 URL 구조를 그대로 따릅니다.

### 모듈 배치 규칙

배치 위치는 **"이 모듈을 누가 쓰는가(사용 범위)"**로 결정합니다. 좁은 범위에서 시작해, 공유 범위가 넓어질수록 위로 올립니다.

1. **계층 내 전용 모듈** — 해당 계층의 폴더 안에 배치
   ```
   /products/components/ProductCard.tsx   → /v2/products 에서만 사용
   ```

2. **상위 계층에서 공유** — 공통 조상 계층에 배치
   ```
   /v2/components/SharedButton.tsx        → /v2/* 전체에서 공유
   ```

3. **앱 전역 공유** — 앱 루트의 `ui/components/` 또는 `hooks/`에 배치
   ```
   ui/components/AppTranslator.tsx        → 앱 전체에서 공유
   ```

4. **앱 간 공유** — 공용 패키지에 배치
   ```
   {{packages.ui}}/components/Button.tsx  → 모든 신규 앱에서 공유
   ```

> **왜 그렇게 하는가**: 사용 범위와 폴더 깊이를 일치시키면, "이 컴포넌트를 고치면 어디까지 영향이 가는가"를 경로만 보고 알 수 있다. 전용 모듈을 전역에 두면 전역이 비대해지고, 공유 모듈을 특정 계층에 두면 중복/우회 import가 생긴다.

---

## 6. 폴더 네이밍 컨벤션

### 필수 폴더명 (통일 필요)

각 계층에서 모듈을 구성할 때 반드시 다음 폴더명을 사용합니다. (어느 앱/계층을 열어도 같은 자리에 같은 종류의 파일이 있게 하기 위함)

| 폴더명        | 용도                                  | 예시 파일                                     |
| ------------- | ------------------------------------- | --------------------------------------------- |
| `components/` | UI 컴포넌트                           | `ProductCard.tsx`, `OrderTable.tsx`           |
| `schemas/`    | Zod 유효성 검사 스키마                | `productSchema.ts`, `orderSchema.ts`          |
| `hooks/`      | Custom React Hooks (queries 제외)     | `useProduct.ts`, `useOrder.ts`                |
| `queries/`    | 서버 상태 라이브러리 기반 데이터 조회 훅 | `useProductList.ts`, `useProductSellers.ts`   |
| `mutations/`  | 서버 상태 라이브러리 기반 데이터 변경 훅 | `useApproveProduct.ts`, `useUpdateProduct.ts` |
| `mappers/`    | API ↔ UI 타입 변환 함수               | `productListMapper.ts`, `productStatusMapper.ts` |
| `types/`      | TypeScript 타입 정의                  | `product.types.ts`, `order.types.ts`          |
| `utils/`      | 유틸리티 함수                         | `formatDate.ts`, `parsePrice.ts`              |
| `constants/`  | 상수 정의                             | `productStatus.ts`, `orderStatus.ts`          |

### Barrel Export (index.ts)

각 모듈 폴더에는 **반드시 `index.ts`** barrel 파일을 만들어 export를 통합합니다.

```
products/
├── components/
│   ├── index.ts              # export { ProductList } from './ProductList'
│   ├── ProductList.tsx
│   └── ProductTable.tsx
├── queries/
│   ├── index.ts              # export { useProductList } from './useProductList'
│   └── useProductList.ts
├── mutations/
│   ├── index.ts
│   └── useAddProduct.ts
└── mappers/
    ├── index.ts
    └── productListMapper.ts
```

```typescript
// O index.ts를 통해 폴더 단위로 import
import { ProductList } from './components'
import { useProductList } from './queries'

// X 개별 파일을 직접 import (index.ts가 있는 경우)
import { ProductList } from './components/ProductList'
```

**하위 폴더가 있는 경우** (예: `components/layout/`, `components/list/`)는 하위 폴더 barrel을 상위 barrel에서 re-export합니다.

```typescript
// components/layout/index.ts
export { AppLayout } from './AppLayout'
export { AppSidebar } from './AppSidebar'

// components/index.ts — 하위 폴더를 re-export
export * from './layout'
export * from './list'
export * from './detail'
```

> **왜 그렇게 하는가**: barrel을 통하면 import 경로가 폴더 단위로 안정화되어, 파일을 옮기거나 분리해도 소비처 import가 깨지지 않는다. (단, API 패키지처럼 순환 참조가 우려되는 곳에서는 barrel 규칙이 달라진다 — 8장 참조.)

### 파일 네이밍 규칙

```typescript
// 컴포넌트: PascalCase
ProductCard.tsx
OrderDetailModal.tsx

// 훅: camelCase with 'use' prefix
useProduct.ts
useOrderList.ts

// 스키마: camelCase with 'Schema' suffix
productSchema.ts
orderFormSchema.ts

// 타입: camelCase with '.types' suffix
product.types.ts
order.types.ts

// 매퍼: 파일명 = 함수명 (예: productListMapper.ts → export function productListMapper())
productListMapper.ts

// 유틸: camelCase
formatDate.ts
parseProductData.ts
```

### 페이지 레벨 모듈 네이밍 규칙 (클라이언트 뷰 네이밍)

`queries/`, `mutations/`, `mappers/` 등 페이지 레벨 모듈의 네이밍은 **API 도메인명이 아닌 클라이언트 뷰 네이밍**을 따릅니다.

```typescript
// API 도메인: restricted-goods (API 모듈명)
// 페이지 뷰: RestrictedProduct (클라이언트에서 사용하는 도메인명)

// O 클라이언트 뷰 네이밍
useRestrictedProductList.ts        // query
useAddRestrictedProduct.ts         // mutation
restrictedProductListMapper.ts     // mapper
RestrictedProductTable.tsx         // component

// X API 도메인 네이밍 그대로 사용
useRestrictedGoodsList.ts
restrictedGoodsListMapper.ts
```

> API 모듈(`api/`)의 함수명은 API 스펙을 그대로 따르되, 페이지 레벨에서는 사용자에게 보이는 도메인 용어로 변환합니다.
> **왜 그렇게 하는가**: 페이지 코드는 "사용자가 보는 화면"의 언어로 읽혀야 한다. API 스펙 용어가 화면 코드까지 새어 나오면, 백엔드 네이밍이 바뀔 때 화면 코드까지 흔들리고 가독성도 떨어진다.

### 앱 내부 UI 컴포넌트 네이밍 규칙

앱 전용 UI 컴포넌트(앱 루트 `ui/components/`)는 **접두사 없이 기능/역할 기반**으로 네이밍합니다.

원칙:

1. **앱 접두사 불필요**: 앱 내부 컴포넌트는 이미 앱 경로로 격리되어 있다.
2. **import 경로가 네임스페이스 역할**: 앱 로컬 alias(`{{apps[].alias}}/ui/components`)와 공용 패키지(`{{packages.uiComponentsImport}}`)가 경로로 구분된다.
3. **기능/역할 기반 네이밍**: 컴포넌트가 하는 일을 명확히 표현한다.

```typescript
// O 올바른 네이밍 — 접두사 없음, 기능 기반
// ui/components/: QueryClientProvider.tsx, Form.tsx, Table.tsx, LinkCard.tsx ...

import { Form } from '<app-alias>/ui/components'      // 앱 전용
import { Button } from '{{packages.uiComponentsImport}}' // 공용 패키지
```

**공용 패키지와 이름이 충돌할 때**는 alias import로 구분합니다.

```typescript
// 공용 패키지에 Form이 있고, 앱에서 확장한 Form이 필요한 경우
import { Form as BaseForm } from '{{packages.uiComponentsImport}}'
import { Form } from '<app-alias>/ui/components' // 앱 전용 확장
```

### 아이콘 컴포넌트 네이밍 규칙

아이콘 컴포넌트는 **`Icon` 접두사** 패턴을 사용합니다. (`Icon` + PascalCase, 파일명도 동일)

```typescript
// O 올바른 네이밍
function IconHourglass() { ... }
function IconArrowRight() { ... }

// X 잘못된 네이밍 (접두사가 접미사로)
function HourglassIcon() { ... }
```

### 계층별(app/) 컴포넌트 네이밍 규칙

`app/` 하위 계층의 컴포넌트(`app/**/components/`)도 **앱 접두사 없이** 네이밍하되, **도메인 컨텍스트 + 역할** 기반으로 이름 짓고 **파일명 = 컴포넌트명**을 일치시킵니다.

1. **앱 접두사 불필요**: import 경로(앱 alias)가 이미 네임스페이스 역할.
2. **도메인 컨텍스트 유지**: 제네릭한 이름(`List`, `Form`)을 피하고 도메인을 포함 (`ProductList`, `RegisterForm`).
3. **파일명 = 컴포넌트명**: `ProductBasicInfoForm` 컴포넌트 → `ProductBasicInfoForm.tsx` 파일.

```
app/v2/products/components/
├── ProductList.tsx              # O 도메인(Product) + 역할(List)
├── ProductFilter.tsx
└── ProductBasicInfoForm.tsx

app/register/components/
├── RegisterForm.tsx             # O 도메인(Register) + 역할(Form)
└── RegisterAgreement.tsx

app/components/
├── HeroSection.tsx              # O 역할 기반 (루트 계층)
└── CTASection.tsx
```

### 스타일링 컨벤션 (`{{styling.system}}`)

모든 컴포넌트의 스타일은 **하나의 `style` 객체 패턴**으로 작성합니다.

1. **하나의 `style` 객체에 집약**: 컴포넌트별로 `const style = { ... }` 형태.
2. **각 키는 요소 역할을 나타내는 이름**: `container`, `header`, `title` 등.
3. **개별 변수 분산 금지**: `const containerStyle = ...` 형태로 흩뿌리지 않음.

```typescript
// 예시는 Panda CSS 문법이지만, 핵심은 "style 객체 집약" 패턴이며
// 스타일 함수 import 경로는 프로젝트 설정값({{apps[].styledSystemImport}})을 따른다.
import { css } from '<styled-system-import>/css'

export function RegisterForm() {
  return (
    <div className={style.container}>
      <h1 className={style.title}>제목</h1>
      <p className={style.subtitle}>부제목</p>
    </div>
  )
}

// O style 객체에 집약
const style = {
  container: css({ display: 'flex', flexDirection: 'column', gap: '24px' }),
  title: css({ textStyle: 'Title_0', color: 'Text.Primary' }),
  subtitle: css({ textStyle: 'Body_2', color: 'Text.Tertiary' }),
}

// X 개별 변수로 분산 (사용 금지)
const containerStyle = css({ display: 'flex' })
const titleStyle = css({ textStyle: 'Title_0' })
```

#### 시멘틱 토큰 우선 사용 (스타일 철학)

`{{styling.semanticTokens}}`가 켜진 프로젝트에서는, 디자인 소스(예: Figma)가 **시멘틱 토큰**(`var(--bg/primary)`, `var(--text/tertiary)` 등)을 제공할 때 코드에서도 **베이스 토큰(`Gray.100`) 대신 시멘틱 토큰(`Text.Primary`)을 사용**합니다.

- 색상: `backgroundColor: 'Bg.Secondary'`(= `var(--bg/secondary)`), `color: 'Text.Primary'`(= `var(--text/primary)`) — 베이스 `Gray.*` 직접 사용 지양.
- 반경(radius): 시멘틱 반경 키(`'L'`, `'XXL'`, `'S'` 등) 사용 — `'12px'` 같은 px 직접 사용 지양.
- 간격(spacing): 토큰 키(`'40'`, `'20'`, `'16'` 등) 참조 — 토큰이 있는 값은 px 대신 키 사용. **토큰이 없는 값은 고정 px 허용.**

> 디자인 소스에 시멘틱 토큰이 없고 고정값만 있으면 베이스 토큰이나 px 값을 사용해도 됩니다.
> 구체 토큰명(`Text.Primary`, `'L'`, `'40'` 등)과 그 매핑 표는 **프로젝트 토큰 정의**(`{{styling.tokenDocSection}}`)를 기준으로 하며, 위 이름은 예시입니다.
> **왜 그렇게 하는가**: 시멘틱 토큰은 "의미"(이 색은 본문 텍스트다)를 가리키므로, 다크모드/리브랜딩 시 토큰 매핑만 바꿔도 전 화면이 일관되게 따라온다. 베이스 토큰/px를 직접 박으면 의미가 사라져 일괄 변경이 불가능하다.

> 레거시 앱이 다른 스타일 시스템(예: CSS Modules + SCSS의 `token()` 함수)을 쓰는 경우에도 **"시멘틱 변수가 있으면 시멘틱 토큰을 쓴다"**는 철학은 동일합니다. 구체 문법/매핑은 해당 앱의 스타일 시스템 규칙을 따릅니다.

---

## 7. 컴포넌트 재사용 가이드

### 핵심 원칙

신규(tier=new) 앱에서 UI를 구현할 때는 **공용 UI 패키지(`{{packages.ui}}`)의 컴포넌트를 최우선**으로 사용한다. 네이티브 HTML 태그(`<button>`, `<input>`, `<p>` …)를 직접 스타일링해 쓰지 않는다.

> **왜 그렇게 하는가**: 디자인 시스템 컴포넌트를 거치면 색·간격·상태(hover/disabled/error) 처리가 한 곳에서 관리된다. 네이티브 태그를 직접 꾸미면 화면마다 미묘하게 달라지고, 리브랜딩·접근성 개선이 전수 수정으로 번진다.

### 공용 컴포넌트 우선 — "진실의 원천은 패키지 export"

구현 전에 `{{packages.uiComponentsImport}}`가 **실제로 export하는 컴포넌트**를 먼저 확인한다. 대부분의 디자인 시스템은 다음 범주를 제공한다(컴포넌트명·props는 프로젝트마다 다르므로 **문서가 아니라 패키지 export를 기준**으로 삼는다):

- 버튼류 / 입력류(텍스트·선택·체크·라디오·토글·날짜) / 텍스트(타이포 컴포넌트) / 레이아웃(폼 컨테이너·그리드·섹션·구분선) / 데이터 표시(테이블·뱃지) / 네비게이션 / 기타(스피너·셀렉터 등)

```typescript
// ❌ 네이티브 태그 직접 사용
<button onClick={handleLogin}>로그인</button>
<p style={{ fontSize: '18px', fontWeight: 'bold' }}>제목</p>
<input type="email" placeholder="이메일" />

// ✅ 공용 UI 패키지 컴포넌트 사용 (컴포넌트명은 예시 — 실제 export 확인)
<Button tone="primary" onClick={handleLogin}>로그인</Button>
<Text type="title">제목</Text>
<TextInput type="email" placeholder="이메일" />
```

> 복합 컴포넌트는 이중 래핑에 주의한다. 예를 들어 라벨/필드 래퍼가 내장된 컴포넌트(날짜 범위 입력 등)를 다시 라벨 래퍼로 감싸면 중복된다. 패키지가 제공하는 "완성형" 컴포넌트가 있으면 하위 요소를 직접 조립하지 말고 그것을 쓴다.

### 새 컴포넌트를 만들 때: 배치 판단 (재사용 범위 기준)

공용 패키지·앱 전역에 없을 때만 새로 만든다. 배치는 **재사용 범위**가 결정한다(5장·10장과 동일 원칙):

1. **한 페이지/기능에서만** → 그 계층 폴더의 `components/`
   예: `{{monorepo.appsDir}}/<app>/app/<route>/components/DashboardCard.tsx`
2. **한 앱의 여러 페이지에서** → 앱 전역 `ui/components/`
   예: `{{monorepo.appsDir}}/<app>/ui/components/SocialLoginButton.tsx`
3. **2개 이상 앱에서** → 공용 UI 패키지(`{{packages.ui}}`)에 추가 검토 (디자인 시스템 일관성 필요 시)

### 체크리스트

UI 요소 구현 전 확인:

- [ ] `{{packages.ui}}`에 쓸 수 있는 컴포넌트가 있는가? (export를 직접 확인)
- [ ] 없다면 앱 전역 `ui/components/`에 있는가?
- [ ] 그래도 없다면: 일회성 → 페이지 `components/`, 재사용 → 앱 `ui/components/`, 범용 → 공용 패키지 검토

---

## 8. API 설계 가이드

이 장이 "신규 앱 개발 방식"의 핵심이다. 아래 역할 분리를 지키면 API 코드가 어느 프로젝트에서든 동일한 형태로 정착한다.

### 핵심 원칙

1. API 호출 함수와 요청/응답 타입은 **공용 API 패키지(`{{packages.api}}`)**에서 관리한다.
2. 앱 로컬 `api/`에는 **클라이언트 인스턴스 설정만** 둔다(팩토리로 인스턴스 생성).
3. **서버 컴포넌트**는 API를 직접 호출할 수 있다.
4. **클라이언트 컴포넌트**는 반드시 서버상태 라이브러리(`{{api.queryLib}}`)를 통해 호출한다.
5. 클라이언트 컴포넌트의 읽기 API는 **페이지 레벨 mapper 함수**로 View Data로 가공해 쓴다. mapper와 View 타입은 페이지 `mappers/`에 두고 **API 모듈에는 두지 않는다.**

> **왜 그렇게 하는가**: API 함수/타입을 공용 패키지에 모으면 여러 앱이 같은 엔드포인트를 재사용한다. mapper를 페이지로 분리하면, 같은 API라도 화면마다 다른 View로 가공할 수 있고 API 계약(서버 응답)과 화면 표현이 분리되어 한쪽이 바뀌어도 다른 쪽이 안 깨진다.

### 공용 API 패키지 구조

```
{{monorepo.packagesDir}}/api/                 # 공용 API 패키지 ({{packages.api}})
├── types.ts                                   # 공통 타입 (ApiResponse 등)
├── index.ts                                   # 통합 export
└── <domain>/                                  # 도메인별 ({{api.domains}})
    ├── index.ts                               # barrel export
    ├── client.ts                              # 클라이언트 인스턴스 팩토리
    ├── server.ts                              # 서버 인스턴스 팩토리
    └── v1/<resource>/                         # 엔드포인트 함수 + 타입
```

### export 구조 (순환 참조 방지)

각 리프 폴더의 `index.ts`는 **자기 엔드포인트 함수 + 자기 `types.ts`만** export한다. 도메인 barrel(`<domain>/index.ts`)에서 **모든 리프 폴더를 평탄하게 직접 나열**해 export한다.

```typescript
// {{monorepo.packagesDir}}/api/<domain>/v1/orders/index.ts
export async function getOrders(api /* client 인스턴스 */, params) { ... }
export * from './types'          // 자기 types.ts만
// ❌ export * from './[orderId]' ← 하위 폴더 re-export 금지 (순환 참조)

// {{monorepo.packagesDir}}/api/<domain>/index.ts  (barrel)
export * from './v1/orders'
export * from './v1/orders/[orderId]'
export * from './v1/products'
// ... 모든 리프 폴더를 평탄하게 나열
```

> **왜 평탄하게 나열하나**: 중간 폴더 index가 하위 폴더를 re-export하면 깊은 트리에서 순환 참조가 생기기 쉽다. barrel 한 곳에서 평탄하게 나열하면 의존 방향이 단순해진다.

### 팩토리 패턴 + 앱 로컬 인스턴스

공용 패키지는 클라이언트를 직접 만들지 않고 **설정을 주입받는 팩토리**를 제공한다. 각 앱이 자기 설정으로 인스턴스를 만든다.

```typescript
// {{packages.api}}/<domain>/client.ts  (팩토리 — 공용 패키지가 제공, 예시는 ky 기준)
import ky from '{{packages.lib}}/ky'
export function createClientApi(config) {
  return ky.create({
    prefixUrl: config.prefixUrl,
    hooks: { beforeRequest: [(req) => { const t = config.getToken(); t && req.headers.set('Authorization', String(t)) }] },
  })
}

// {{monorepo.appsDir}}/<app>/api/<domain>/client.ts  (앱 전용 인스턴스)
import { createClientApi } from '{{packages.api}}/<domain>/client'
import { getCookie } from '{{packages.lib}}/cookies-next/client'
export const clientApi = createClientApi({ prefixUrl: '/api/<domain>', getToken: () => getCookie('token') })
```

### API 호출 함수 + 타입 (공용 패키지)

API 함수는 **client 인스턴스를 첫 번째 인자로** 받는다(`{{api.clientArgFirst}}` 컨벤션). 요청/응답 타입은 각 엔드포인트 `types.ts`에 둔다(`{{api.schemaLib}}` 예: zod).

```typescript
// {{packages.api}}/<domain>/v1/sellers/index.ts
import { ApiResponse } from '{{packages.api}}/types'
import type { GetSellersResponseData } from './types'
export async function getSellers(api) {
  return api.get('v1/sellers').json<ApiResponse<GetSellersResponseData>>()
}
```

### 페이지 레벨 Mapper (`app/**/mappers/`)

Raw Data → View Data 변환은 그 API를 쓰는 페이지의 `mappers/`가 담당한다.

```typescript
// app/<route>/mappers/sellersMapper.ts
import type { GetSellersResponseData } from '{{packages.api}}/<domain>'
export type SellerView = { id: number; name: string }
export function sellersMapper(data: GetSellersResponseData): SellerView[] {
  return data.map((s) => ({ id: s.sellerId, name: s.sellerName }))
}
```

### 앱에서 사용하기

```typescript
// 클라이언트 컴포넌트: 서버상태 라이브러리 + select(mapper) + clientApi 전달
// app/<route>/queries/useSellers.ts
import { useQuery } from '{{packages.lib}}/tanstack/react-query'
import { getSellers } from '{{packages.api}}/<domain>'
import { clientApi } from '{{app.alias}}/api/<domain>/client'
import { sellersMapper } from '../mappers/sellersMapper'
export function useSellers() {
  return useQuery({
    queryKey: ['sellers'],
    queryFn: () => getSellers(clientApi),
    select: (res) => sellersMapper(res.data),
  })
}

// 서버 컴포넌트: 서버 인스턴스로 직접 호출
// app/<route>/page.tsx
import { getSellers } from '{{packages.api}}/<domain>'
import { createServerApi } from '{{app.alias}}/api/<domain>/server'
export default async function Page() {
  const serverApi = await createServerApi()
  const res = await getSellers(serverApi)
  return <SellerList sellers={res.data} />
}
```

### 서버 vs 클라이언트 컴포넌트 규칙

| 구분 | API 직접 호출 | 서버상태 라이브러리 | mapper |
| --- | :---: | :---: | :---: |
| 서버 컴포넌트 | ✅ 가능 | ❌ 불필요 | ✅ 권장 |
| 클라이언트 컴포넌트 | ❌ 금지 | ✅ 필수 | ✅ 필수 |

### 역할 분리

| 위치 | 포함 | 미포함 |
| --- | --- | --- |
| `{{packages.api}}` | API 호출 함수, Raw 요청/응답 타입, 클라이언트 팩토리 | 앱 전용 설정, View 타입, mapper |
| 앱 `api/` | 팩토리로 만든 클라이언트 인스턴스(앱 전용 설정) | API 호출 함수, 타입 |
| 페이지 `mappers/` | View 타입, mapper 함수(Raw → View) | API 호출 함수 |
| 페이지 `queries/` | 서버상태 훅(queryFn + select에서 mapper 호출) | - |

### API 설계 체크리스트

- [ ] API 호출 함수가 `{{packages.api}}`에 있는가?
- [ ] API 함수가 client 인스턴스를 첫 번째 인자로 받는가?
- [ ] 앱 로컬 `api/`에는 인스턴스 설정만 있는가?
- [ ] 요청/응답 타입이 `{{packages.api}}` 내부 `types.ts`에 있는가?
- [ ] API 모듈에 View 타입/select 함수가 없는가? (있으면 페이지 `mappers/`로)
- [ ] 클라이언트 컴포넌트가 서버상태 라이브러리 + select(mapper)를 쓰는가?

---

## 9. 폼 검증 패턴 (Zod + RHF)

폼 입력이 있는 UI는 **스키마 기반 검증 라이브러리(`{{api.schemaLib}}`, 예: Zod) + React Hook Form**으로 구현한다.

| 요소 | 위치 | 역할 |
| --- | --- | --- |
| 스키마 | `schemas/xxxSchema.ts` | 유효성 규칙 + 에러 메시지 |
| useForm | 컴포넌트 내부 | 폼 상태 + resolver 연결 |
| Controller | JSX 내부 | 폼 필드 ↔ UI 컴포넌트 연결 |
| error prop | 공용 UI 컴포넌트 | 에러 메시지 인라인 표시 |

```typescript
// schemas/addItemSchema.ts
import { z } from '{{packages.lib}}/zod'
export const addItemSchema = z.object({
  ids: z.string().min(1, 'ID를 입력해 주세요'),
})
export type AddItemFormValues = z.infer<typeof addItemSchema>
export const addItemDefaultValues: AddItemFormValues = { ids: '' }

// 컴포넌트
import { useForm, Controller } from '{{packages.lib}}/react-hook-form'
import { zodResolver } from '{{packages.lib}}/hookform/resolvers/zod'
const { control, handleSubmit } = useForm({
  resolver: zodResolver(addItemSchema),
  defaultValues: addItemDefaultValues,
  mode: 'onChange',
})
<Controller name="ids" control={control} render={({ field, fieldState }) => (
  <Textarea {...field} error={fieldState.error?.message} />   // 에러 → UI 자동 표시
)} />
```

### 핵심 원칙

1. **에러 메시지는 스키마에 정의** — UI 컴포넌트에 직접 쓰지 않는다.
2. **`error` prop으로 전달** — 공용 UI 입력 컴포넌트가 에러 스타일을 자동 적용한다.
3. **수동 파싱/검증 금지** — `.refine()` / `.transform()`을 활용한다.
4. **스키마 파일에 defaultValues도 함께** — 타입과 초기값을 한 곳에서 관리.

> **왜 그렇게 하는가**: 검증 규칙·메시지·초기값·타입을 스키마 한 곳에 모으면 폼이 늘어나도 일관되고, UI 컴포넌트는 "표시"에만 집중한다. 수동 `if` 검증은 빠뜨리기 쉽고 메시지가 흩어진다.

---

## 10. 모듈 배치 의사결정 가이드

새 모듈(컴포넌트·훅·유틸 등)을 만들 때:

```
새 모듈 생성 필요
   │
   ▼  2개 이상 앱에서 쓰나? ──Yes──▶ 공용 패키지({{packages.scope}}/*)에 추가
   │No
   ▼  한 앱 안에서 앱 전역으로 쓰나? ──Yes──▶ 앱의 ui/ · hooks/ 등에 배치
   │No
   ▼  특정 pathname 계층에서만 쓴다 ──▶ 그 계층 폴더 안 components/ · hooks/ 등에 배치
```

### 예시 시나리오 (도메인명은 예시)

- 상품 상세에서만 쓰는 갤러리 → `app/<route>/products/[productId]/components/ImageGallery.tsx`
- 상품 영역 전체가 쓰는 포맷터 → `app/<route>/products/utils/formatPrice.ts`
- 앱 전역 번역 컴포넌트 → `ui/components/Translator.tsx`
- 여러 앱이 쓰는 버튼 → `{{monorepo.packagesDir}}/ui/components/Button.tsx`

---

## 11. 신규 앱 개발 시 주의사항

### Must Do

1. **공용 패키지(`{{packages.scope}}/*`)만 사용** — 외부 라이브러리는 `{{packages.lib}}`를 통해 import한다.

   ```typescript
   // ✅ import { useQuery } from '{{packages.lib}}/tanstack-react-query'
   // ❌ import { useQuery } from '@tanstack/react-query'   // 직접 import 금지
   ```

2. **계층별 폴더 구조 준수** — 계층 전용 모듈은 그 계층 폴더 안에, 폴더명은 컨벤션(components/schemas/hooks/…) 통일.

3. **참조 앱 패턴 따르기** — 새 기능은 참조 앱(`{{migration.referenceApp}}`)의 동일 기능 구현을 먼저 확인하고 같은 패턴을 유지한다.

### Must Not

1. **레거시 패키지 직접 사용 금지** (해당 프로젝트에 레거시가 있는 경우).
2. **계층 무시 금지** — 한 페이지 전용 컴포넌트를 전역에 두거나, 공용 컴포넌트를 한 계층에 가두지 않는다.
3. **외부 라이브러리 직접 import 금지** — 필요하면 `{{packages.lib}}`에 먼저 추가한다.

### 참조 앱 구조 예시 (tier=new)

```
{{monorepo.appsDir}}/<app>/
├── app/<route>/
│   ├── page.tsx
│   ├── components/      # 이 계층 전용 컴포넌트
│   ├── schemas/         # 이 계층 전용 스키마
│   ├── queries/         # 서버상태 훅
│   ├── mappers/         # View 타입 + mapper
│   └── [id]/ ...        # 하위 계층 (자기 components/ 보유)
├── api/<domain>/client.ts   # 클라이언트 인스턴스 설정만
├── ui/components/           # 앱 전역 UI
├── hooks/                   # 앱 전역 훅
└── constant/                # 앱 전역 상수
```

---

## 12. 요약 체크리스트

신규 개발 시 확인:

- [ ] 공용 패키지(`{{packages.scope}}/*`)만 사용하는가? (외부 라이브러리는 `{{packages.lib}}` 경유)
- [ ] 모듈이 재사용 범위에 맞는 계층에 배치됐는가?
- [ ] 폴더명이 컨벤션을 따르는가? (components, schemas, queries, mappers, hooks …)
- [ ] 클라이언트 데이터 조회가 서버상태 라이브러리 + select(mapper) 패턴인가?
- [ ] 참조 앱(`{{migration.referenceApp}}`)의 동일 기능 패턴을 참고했는가?
- [ ] 2개 이상 앱에서 공유할 모듈은 공용 패키지에 있는가?
