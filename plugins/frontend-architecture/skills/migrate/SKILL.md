---
name: migrate
description: 레거시 앱의 코드를 프로젝트의 신규 공유 패키지 패턴으로 변환합니다. "/migrate", "마이그레이션해줘", "레거시 코드 변환", "신규 패턴으로 바꿔줘" 요청에 사용하세요. 레거시 import를 공유 lib/ui/api 패키지로 교체하거나 기존 CSS를 프로젝트 스타일 시스템으로 변환할 때 사용합니다.
argument-hint: '<target-path>'
---

레거시 코드를 신규 공유 패키지 패턴으로 변환하는 가이드를 제공해줘.
이 스킬은 특정 프로젝트에 고정돼 있지 않다. **실행 시점에 프로젝트의 컨벤션을 먼저 읽고** 그 매핑을 적용한다.

대상: $ARGUMENTS
예시:

- `/migrate apps/web/app/[locale]/member/login` → 디렉토리 단위 변환
- `/migrate apps/ota/components/Header.tsx` → 특정 파일 변환
- `/migrate apps/web/hooks/useOrder.ts` → 특정 훅 변환

## 0. 컨벤션 로드 (항상 먼저)

1. 프로젝트 루트에서 다음을 읽는다:
   - `ARCHITECTURE.md` — 계층·import·API·스타일링 규칙의 서술적 원천. 변환 결과는 이 문서를 따른다.
   - (폴백) 프로젝트에 `ARCHITECTURE.md`가 없으면 → 플러그인 번들 base `${CLAUDE_PLUGIN_ROOT}/ARCHITECTURE.base.md`를 기본 패턴으로 읽어 따른다. (/frontend-architecture:init 으로 프로젝트 ARCHITECTURE.md를 만들어두면 더 좋다.)
   - `.claude/conventions.json` — `migration`, `packages`, `apps`(tier), `api`, `styling`, `monorepo` 노브.
2. `conventions.json`이 없으면 `package.json`(workspaces)·폴더 구조·`{{migration.referenceApp}}`의 기존 코드에서 추론하고, 애매하면 **추측하지 말고 사용자에게 물어본다.**
3. 아래에서 `{{...}}`는 컨벤션에서 읽은 값을 의미한다:
   - `{{packages.lib}}` / `{{packages.ui}}` / `{{packages.uiComponentsImport}}` / `{{packages.api}}` = 신규 공유 패키지 import 경로
   - `{{packages.scope}}` = 공유 패키지 네임스페이스 (예: `@org`)
   - `{{app.styledSystemImport}}` = 앱 로컬 스타일 import 경로 (예: `@<앱alias>/ui/styled-system/css`)
   - `{{migration.legacyApps}}` = 레거시 앱 목록 (이 스킬의 대상)
   - `{{migration.referenceApp}}` = 신규 패턴 참조 기준 앱 (예: admin)
   - `{{migration.importMap}}` = 레거시 import → 신규 import 매핑 (아래 변환 표의 기준)
   - `{{appsDir}}` / `{{packagesDir}}` = `monorepo.appsDir` / `monorepo.packagesDir`

> **변환 매핑은 `migration.importMap` + ARCHITECTURE.md를 1차 기준으로 한다. 아래 표는 일반적 예시이며, 프로젝트가 제공한 매핑이 있으면 그것을 따른다.**

---

## 1단계: 대상 코드 분석

1. 대상 파일/디렉토리의 모든 코드를 읽고 import 목록을 추출
2. 레거시 패키지 의존성을 식별하고 사용 횟수를 카운트
3. 사용 중인 패턴 식별 (폼 처리, 데이터 fetching, 스타일링 등)

## 2단계: {{packages.lib}} 존재 여부 확인 (중요!)

레거시 코드에서 사용하는 외부 라이브러리가 `{{packages.lib}}`에 이미 있는지 **반드시 확인**:

```bash
# {{packagesDir}}/lib/package.json 의 exports 필드 확인
```

`{{packages.lib}}`의 실제 `exports`에서 사용 가능한 서브경로(예: `dayjs`, `react-hook-form`, `zod`, `hookform/resolvers/zod`, `tanstack/react-query`, `ky`, `cookies-next/client|server`, `dnd-kit/*`, `swiper/*`, `nanoid`, `idb` 등)를 직접 읽어 매핑한다. **고정 목록을 가정하지 말고 실제 exports를 확인할 것.**

### {{packages.lib}}에 없는 라이브러리를 발견한 경우

**바로 추가하면 안 됨!** 아래 기준으로 판단:

1. **참조 앱(`{{migration.referenceApp}}`)에서 같은 라이브러리를 사용하고 있는지 확인** → 쓰고 있다면 `{{packages.lib}}`에 추가 대상
2. **다른 신규(tier=new) 앱에서도 필요할 것 같은지 확인** → 2개 이상 앱에서 사용 예상되면 `{{packages.lib}}`에 추가 대상
3. **해당 앱에서만 쓰는 라이브러리라면** → 앱의 package.json에 직접 추가 (앱 전용 의존성)
4. 추가가 필요한 경우 `/add-lib` 스킬 사용을 안내

---

## 3단계: 변환 매핑 (Before → After)

### 패키지 import 변환

`migration.importMap`을 기준으로 매핑한다. importMap에 없는 항목은 ARCHITECTURE.md 규칙 + 아래 일반 예시로 보강:

| 레거시                             | 신규 (importMap/ARCHITECTURE.md 기준)            | 비고                                               |
| ---------------------------------- | ------------------------------------------------ | -------------------------------------------------- |
| `import from 'utils'`              | 기능별 분리 (대부분 `{{packages.lib}}`)          | 아래 상세 참조                                     |
| `import from 'ui'` / `'legacy-ui'`       | `{{packages.uiComponentsImport}}`                | 컴포넌트명이 다를 수 있음, 확인 필요               |
| `import from 'types'` / `'types2'` | `{{packages.scope}}/types`                       | 공통 타입만. 나머지는 앱 내부 types/ 또는 API types.ts |
| `import from 'constant'`           | `{{packages.scope}}/constant`                    |                                                    |
| `import from 'rest'`               | `{{packages.api}}` 레이어 재작성                 | 아래 상세 참조                                     |
| `import from 'mappers'`            | API select 함수 또는 앱 내부 mappers/            |                                                    |
| `import from 'models'`             | 앱 내부 types/ 또는 API types.ts                 |                                                    |
| `import from 'i18n-legacy'`          | 앱 내부 i18n 설정                                | 신규 앱에서의 i18n 전략에 따라 다름                |
| `import from 'layout'`             | `{{packages.uiComponentsImport}}` 또는 앱 내부 ui/ |                                                  |
| `import from 'logger'`             | 앱 내부 lib/                                      |                                                    |

### utils 패키지 상세 분리

`utils` 류 패키지는 다양한 기능이 섞여 있으므로 기능별로 분리:

```typescript
// BEFORE (레거시)
import FetchServer from 'utils/fetch-server'
import { ssrAllSettled } from 'utils/fetcher'
import { getLocaleString, getCurrency } from 'utils/cookies'
import useGlobalLoading from 'utils/hooks/useGlobalLoading'
import { Image, useRouter } from 'utils/share'

// AFTER (신규)
// fetch → {{packages.lib}}/ky 기반 API 클라이언트로 대체
import { clientApi } from '@<앱alias>/api/<도메인>/client'
// cookies → {{packages.lib}}/cookies-next
import { getCookie } from '@org/lib/cookies-next/client' // = {{packages.lib}}/cookies-next/client
// hooks → 앱 내부 hooks/에 재작성
import { useGlobalLoading } from '@<앱alias>/hooks/useGlobalLoading'
// Image → 프레임워크 기본 (next/image 등) 직접 사용
import Image from 'next/image'
// useRouter → 프레임워크 라우터 (next/navigation 등)
import { useRouter } from 'next/navigation'
```

### rest 패키지 → {{packages.api}} 사용

```typescript
// BEFORE (레거시)
import { Internal } from 'rest'
const restService = new FetchServer()
const data = await restService
  .service<DataType[]>(Internal.bannerList(code, locale))
  .catch(() => null)

// AFTER (신규) - {{packages.api}}에 API 모듈이 있는지 먼저 확인
// 없으면 api.pathPattern(예: {{packagesDir}}/api/<도메인>/<버전>/banners/)에 생성

// 1. <api경로>/banners/types.ts
export type BannerListResponseData = {
  /* ... */
}

// 2. <api경로>/banners/index.ts  (client=ky, clientArgFirst 가정)
import type { KyInstance } from '@org/lib/ky' // = {{packages.lib}}/ky
import { ApiResponse } from '@org/api/types' // = {{packages.api}}/types
export async function getBanners(api: KyInstance, code: string) {
  return api.get(`v1/banners/${code}`).json<ApiResponse<BannerListResponseData>>()
}

// 3. 앱에서 사용 - 클라이언트 인스턴스를 첫 번째 인자로 전달 (api.clientArgFirst)
import { getBanners } from '@org/api/internal' // = {{packages.api}}/<도메인>
import { clientApi } from '@<앱alias>/api/internal/client'

// 서버 컴포넌트
const serverApi = await createServerApi()
const data = await getBanners(serverApi, code)

// 클라이언트 컴포넌트 (api.queryLib)
import { useQuery } from '@org/lib/tanstack/react-query' // = {{packages.lib}}/tanstack/react-query
const { data } = useQuery({
  queryKey: ['banners', code],
  queryFn: () => getBanners(clientApi, code),
})
```

### 폼 처리 변환

```typescript
// BEFORE (레거시) - useState 수동 관리
const [email, setEmail] = useState('')
const [password, setPassword] = useState('')
const handleSubmit = async (e: FormEvent) => {
  e.preventDefault()
  await restService.service(Next.login(email, password))
}
<Input value={email} onChange={setEmail} />

// AFTER (신규) - React Hook Form + Zod (모두 {{packages.lib}} 경유)
import { useForm, Controller } from '{{packages.lib}}/react-hook-form'
import { z } from '{{packages.lib}}/zod'
import { zodResolver } from '{{packages.lib}}/hookform/resolvers/zod'

const schema = z.object({
  email: z.string().email(),
  password: z.string().min(1),
})
const { control, handleSubmit } = useForm({
  resolver: zodResolver(schema),
})
<Controller
  name="email"
  control={control}
  render={({ field, fieldState }) => (
    <TextInput {...field} error={fieldState.error?.message} />
  )}
/>
```

### 스타일링 변환

> `styling.system`에 맞춰 변환한다. 아래는 panda 예시. tailwind/emotion 등이면 해당 시스템 규칙으로 대체.

```typescript
// BEFORE (레거시) - CSS Modules
import style from './Form.module.scss'
<div className={style.wrapper}>

// AFTER (신규) - 앱 로컬 스타일 시스템 ({{app.styledSystemImport}})
import { css, cx } from '@<앱alias>/ui/styled-system/css'
<div className={css({ display: 'flex', padding: '40px 20px' })}>

// 또는 변수로 분리 (시멘틱 토큰 사용 — ARCHITECTURE.md 시멘틱 토큰 규칙 참조, styling.semanticTokens)
const wrapperStyle = css({
  display: 'flex',
  padding: '40 20',
  textStyle: 'Title_2_SB',
  color: 'Text.Primary',
})
<div className={wrapperStyle}>
```

### UI 컴포넌트 변환

```typescript
// BEFORE (레거시)
import { CheckBox, Input, Password } from 'ui'
import MoPageHeader from 'ui/components/layout/MoPageHeader'

// AFTER (신규) - {{packages.uiComponentsImport}}에서 동일 컴포넌트 확인 후 매핑
import { Checkbox, TextInput } from '@org/ui/components' // = {{packages.uiComponentsImport}}
// 앱 전용 레이아웃은 앱 내부 ui/에 작성
import { PageHeader } from '@<앱alias>/ui/components'
```

> **주의**: 레거시 `ui` 패키지와 `{{packages.ui}}`의 컴포넌트명/Props가 다를 수 있음.
> 반드시 `{{packages.uiComponentsImport}}`의 실제 export를 확인하고 매핑할 것.

---

## 4단계: 출력 형식

```markdown
## 마이그레이션 분석

### 레거시 의존성 목록

| 패키지 | import 경로 | 사용 횟수 | 변환 대상 |
| ------ | ----------- | --------- | --------- |

### 변환 계획

각 의존성별 before/after 코드와 변환 이유

### {{packages.lib}} 추가 필요 여부

- 추가 필요한 라이브러리가 있는지, 있다면 근거 (다른 앱에서의 사용 여부)

### 주의사항

- 컴포넌트 Props 차이
- 동작 변경이 예상되는 부분
- 테스트가 필요한 부분

### 예상 변환 결과

- 변환 후 전체 코드 미리보기
```

변환을 직접 실행할지, 가이드만 제공할지 물어봐줘.
