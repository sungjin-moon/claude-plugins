---
name: new-api
description: 'API 패키지에 새 API 모듈(types.ts, index.ts)을 생성합니다. "/new-api", "API 모듈 만들어줘", "새 API 추가해줘", "API 패키지에 엔드포인트 추가" 요청에 사용하세요. 기존 API 수정이 아닌 신규 모듈 생성에 사용합니다.'
argument-hint: '<domain> <resource> <version>'
---

프로젝트의 API 설계 가이드에 따라 공유 API 패키지에 새로운 API 모듈을 생성해줘.
이 스킬은 특정 프로젝트에 고정돼 있지 않다. **실행 시점에 프로젝트의 컨벤션을 먼저 읽고** 그대로 따른다.

인자: $ARGUMENTS
형식: `<도메인> <리소스경로> <버전>`
예시 (도메인은 `conventions.json`의 `api.domains`에서 읽음):

- `/new-api <domain> goods/categories v1`
- `/new-api <domain> links v1`

## 0. 컨벤션 로드 (항상 먼저)

1. 프로젝트 루트에서 다음을 읽는다:
   - `ARCHITECTURE.md` — API 설계 가이드·계층 규칙·네이밍의 **서술적 원천**. 이 문서가 아래 가정과 충돌하면 **항상 프로젝트 문서가 우선**한다.
   - (폴백) 프로젝트에 `ARCHITECTURE.md`가 없으면 → 플러그인 번들 base `${CLAUDE_PLUGIN_ROOT}/ARCHITECTURE.base.md`를 기본 패턴으로 읽어 따른다. (/frontend-architecture:init 으로 프로젝트 ARCHITECTURE.md를 만들어두면 더 좋다.)
   - `.claude/conventions.json` — `packages`, `api`, `monorepo` 노브
2. `conventions.json`이 없으면 `package.json`(workspaces)·기존 API 패키지 구조에서 추론하고, 애매하면 **추측하지 말고 사용자에게 물어본다.**
3. 아래에서 `{{...}}`는 컨벤션에서 읽은 값을 의미한다:
   - `{{packages.api}}` = API 함수 패키지 (예: `@org/api`)
   - `{{packages.lib}}` = 외부 라이브러리 래퍼 패키지 (예: `@org/lib`)
   - `{{monorepo.packagesDir}}` = 공유 패키지 디렉토리 (기본 `packages`)
   - `{{api.pathPattern}}` = 모듈 배치 규칙 (예: `packages/api/<domain>/<version>`)
   - `{{api.domains}}` = 사용 가능한 도메인 목록 (예: `admin`, `internal`)
   - `{{api.client}}` = HTTP 클라이언트 (예: `ky`) / `{{api.clientArgFirst}}` = 첫 인자로 client 전달 여부
   - `{{api.schemaLib}}` = 유효성 검사 라이브러리 (예: `zod`)
   - `{{api.queryLib}}` = 서버 상태 라이브러리 (예: `tanstack-query`)

> 아래 코드 예시는 `client=ky`, `schemaLib=zod`, `queryLib=tanstack-query` 기준이다. 컨벤션의 값이 다르면 그에 맞춰 치환한다.

## 작업 위임

이 스킬은 `frontend-api-builder` agent에게 작업을 위임합니다 (에이전트가 있는 경우).

```
Task(
  subagent_type="frontend-api-builder",
  prompt="
    도메인: $ARGUMENTS에서 첫 번째 인자 (conventions.json의 api.domains 중 하나)
    리소스 경로: $ARGUMENTS에서 두 번째 인자
    API 버전: $ARGUMENTS에서 세 번째 인자

    프로젝트 루트의 ARCHITECTURE.md + .claude/conventions.json을 먼저 읽고,
    그 API 설계 가이드에 따라 공유 API 패키지({{packages.api}})에 새로운 API 모듈을 생성하세요.
  "
)
```

> `frontend-api-builder` 에이전트가 없으면 이 스킬이 직접 아래 절차를 수행한다.

## frontend-api-builder가 하는 일

1. **기존 패턴 파악**

   - `{{monorepo.packagesDir}}/api/` 디렉토리 구조 확인
   - 동일 도메인(`{{api.domains}}` 중 하나)의 기존 API 모듈 패턴 분석
   - API 패키지 `tsconfig.json`의 self-referencing paths 확인
   - **관찰한 패턴이 이 문서의 가정과 다르면, 관찰한 패턴을 따른다.**

2. **파일 생성**

   - `types.ts`: 스키마(`{{api.schemaLib}}`) + 요청/응답 타입
   - `index.ts`: API 호출 함수 + export
   - 기존 `{{packages.api}}` 패턴과 동일한 구조로 생성

3. **통합**

   - 상위 `index.ts`(barrel)에 새 모듈 re-export 추가

4. **사용 예시 제공**
   - 서버 컴포넌트에서의 사용법 (서버 API 인스턴스 생성 사용)
   - 클라이언트 컴포넌트에서의 사용법 (`{{api.queryLib}}` + client 인스턴스 전달)

## 생성 파일 구조

**폴더 구조는 반드시 URL pathname과 1:1로 일치시킵니다.** 배치 규칙은 `{{api.pathPattern}}`을 따른다.

```
{{monorepo.packagesDir}}/api/<도메인>/<버전>/<리소스>/
├── index.ts    # 해당 경로의 API 함수 (목록 조회, 생성 등)
├── types.ts    # 해당 경로의 타입 정의
├── <sub-path>/          # 정적 sub-path → 별도 폴더
│   ├── index.ts
│   └── types.ts
└── [<param>]/           # 동적 path → [param] 폴더
    ├── index.ts
    └── types.ts
```

### 예시 (도메인은 `{{api.domains}}`에서 치환)

```
# URL: GET /v1/order/affiliate/partner (목록)
# URL: GET /v1/order/affiliate/partner/{partnerId} (상세)
# URL: GET /v1/order/affiliate/partner/{partnerId}/sensitive

{{monorepo.packagesDir}}/api/<domainA>/v1/order/affiliate/partner/
├── index.ts              # 목록 조회
├── types.ts
└── [partnerId]/
    ├── index.ts          # 상세, 수정, 승인, sensitive 등
    └── types.ts

# URL: GET /v1/orders/affiliate/wallet/revenue
# URL: GET /v1/orders/affiliate/wallet/balance
# URL: POST /v1/orders/affiliate/wallet/withdrawal
# URL: DELETE /v1/orders/affiliate/wallet/withdrawal/{id}

{{monorepo.packagesDir}}/api/<domainB>/v1/orders/affiliate/wallet/
├── revenue/
│   ├── index.ts
│   └── types.ts
├── balance/
│   ├── index.ts
│   └── types.ts
└── withdrawal/
    ├── index.ts          # POST (출금 요청)
    ├── types.ts
    └── [id]/
        └── index.ts      # DELETE (출금 취소)
```

하나의 `index.ts`에 서로 다른 sub-path의 함수를 모으지 않습니다.

## API 패턴 (자동 생성)

> 아래는 `client=ky` / `schemaLib=zod` 기준 예시. `{{api.*}}` 값이 다르면 그에 맞춰 작성한다.

### types.ts

```typescript
import { z } from '{{packages.lib}}/zod'

export const GetXxxRequestParamsSchema = z.object({ ... })
export type GetXxxRequestParams = z.infer<typeof GetXxxRequestParamsSchema>

export type XxxResponseData = { ... }
```

### index.ts

```typescript
import type { KyInstance } from '{{packages.lib}}/ky'
import { ApiResponse } from '{{packages.api}}/types'

export async function getXxx(api: KyInstance, params: GetXxxRequestParams) {
  const response = await api.get('<버전>/<리소스>', {
    searchParams: GetXxxRequestParamsSchema.parse(params),
  })
  return response.json<ApiResponse<XxxResponseData>>()
}

export * from './types'
```

## 앱에서의 사용 예시

```typescript
// 클라이언트 컴포넌트 (queries/)
import { useQuery } from '{{packages.lib}}/tanstack/react-query'
import { getXxx } from '{{packages.api}}/<domain>'
import { clientApi } from '@<app-alias>/api/<domain>/client'

export function useXxx(params: GetXxxRequestParams) {
  return useQuery({
    queryKey: ['xxx', params],
    queryFn: () => getXxx(clientApi, params),
  })
}
```

> `@<app-alias>`는 대상 앱의 로컬 import alias (`conventions.json`의 `apps[].alias`). client 인스턴스 위치/이름은 앱 로컬 `api/` 폴더의 기존 코드를 따른다.

## 주의사항

- API 함수는 `{{monorepo.packagesDir}}/api/` 에 생성 (앱 로컬 `api/`가 아님)
- `{{packages.lib}}`를 통해서만 외부 라이브러리 import
- `{{api.clientArgFirst}}`가 true면 첫 번째 인자로 `api: KyInstance`(또는 해당 client 타입)를 받음 (기본값 없이 필수)
- 공통 타입은 `{{packages.api}}/types`에서 self-referencing으로 import
- `{{api.schemaLib}}` 스키마로 요청 파라미터 검증

## 후속 작업: Postman Collection 동기화

API 모듈 생성/수정이 완료되면 `/sync-postman` 스킬을 실행하여 Postman Collection도 함께 업데이트합니다.

- API 문서 URL은 `conventions.json`의 도메인별 문서 base URL에서 읽음 / 없으면 사용자에게 물어본다.
- 예: `/sync-postman <api-doc-url>` (도메인별 문서 URL은 `sync-postman` 스킬의 매핑 규칙 참조)
