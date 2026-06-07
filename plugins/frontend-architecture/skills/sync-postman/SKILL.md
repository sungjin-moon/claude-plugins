---
name: sync-postman
description: API 문서 URL을 기반으로 API 코드 모듈을 업데이트하고 Postman Collection을 동기화합니다. "/sync-postman", "Postman 컬렉션 만들어줘", "API 문서 URL로 Postman 동기화", "Postman 업데이트해줘", "API 문서 반영해줘", "API 문서 업데이트해줘" 요청에 사용하세요. API 문서 URL이 포함된 요청이면 이 스킬을 사용합니다.
argument-hint: '<api-doc-url> [collection-name/folder-name]'
---

API 문서 URL을 받아서 **API 코드 모듈 반영**과 **Postman Collection 동기화**를 함께 수행합니다.
이 스킬은 특정 프로젝트에 고정돼 있지 않다. **실행 시점에 프로젝트의 컨벤션을 먼저 읽고** 그대로 따른다.

인자: $ARGUMENTS
형식: `<API 문서 URL> [폴더 이름]`
예시:

- `/sync-postman <api-doc-url>`
- `/sync-postman <api-doc-url> > 상품 기초 데이터`

## 0. 컨벤션 로드 (항상 먼저)

1. 프로젝트 루트에서 다음을 읽는다:
   - `ARCHITECTURE.md` — API 설계 가이드·계층/배치 규칙의 **서술적 원천**. 충돌 시 **항상 프로젝트 문서가 우선**한다.
   - `.claude/conventions.json` — `packages`, `api`, `monorepo` 노브
2. `conventions.json`이 없으면 `package.json`·기존 API 패키지 구조에서 추론하고, 애매하면 **추측하지 말고 사용자에게 물어본다.**
3. 아래에서 `{{...}}`는 컨벤션에서 읽은 값을 의미한다:
   - `{{packages.api}}` = API 함수 패키지 (예: `@org/api`)
   - `{{monorepo.packagesDir}}` = 공유 패키지 디렉토리 (기본 `packages`)
   - `{{api.domains}}` = 도메인 목록 (예: `admin`, `internal`)
   - `{{api.pathPattern}}` = 모듈 배치 규칙
4. **Postman/회사 고유값**(workspace ID, collection ID, base URL 변수, API 문서 base URL)은 표준 스키마에 없을 수 있다.
   - `conventions.json`에 `postman` 섹션(또는 `api.docUrls`)이 있으면 그 값을 사용한다.
   - **없으면 사용자에게 물어본다.** 임의로 ID/URL을 발명하지 않는다.

## 기본 매핑 규칙

아래 표의 값들(문서 URL 패턴, Collection 이름·ID, base URL 변수, workspace ID)은 **프로젝트 고유값**이다.
`conventions.json`의 `postman` 섹션에서 읽고, 없으면 사용자에게 확인한다.

| API 문서 URL 패턴 | Collection (이름 / ID) | API 코드 경로 | Base URL 변수 |
|---|---|---|---|
| `<도메인A 문서 base>/**` | `{{postman.collections.<domainA>.name}}` (ID: `{{postman.collections.<domainA>.id}}`) | `{{monorepo.packagesDir}}/api/<domainA>/` | `{{postman.collections.<domainA>.baseUrlVar}}` |
| `<도메인B 문서 base>/**` | `{{postman.collections.<domainB>.name}}` (ID: `{{postman.collections.<domainB>.id}}`) | `{{monorepo.packagesDir}}/api/<domainB>/` | `{{postman.collections.<domainB>.baseUrlVar}}` |

- Workspace ID: `{{postman.workspaceId}}` (conventions.json에서 읽음 / 없으면 사용자에게 질문)
- 위 매핑에 해당하지 않는 URL이면 사용자에게 Collection 이름과 폴더를 확인

## 작업 흐름

### Step 1: API 문서 파싱

WebFetch 도구를 사용하여 주어진 URL의 API 문서를 가져옵니다.

프롬프트:
```
Extract ALL API endpoints from this page. For each endpoint provide:
1. HTTP method (GET, POST, PUT, DELETE, PATCH)
2. Full URL path
3. ALL request parameters (query params for GET, request body fields for POST/PUT with types and descriptions)
4. ALL response body fields with names, types, nested structure, descriptions
5. Description/name of the endpoint (Korean name if available)
Be extremely thorough - list every single endpoint and every single field.
```

### Step 2: 기존 API 코드 비교

1. 해당 API 경로의 기존 `types.ts`, `index.ts` 파일을 읽는다
2. API 문서에서 파싱한 엔드포인트/필드와 비교하여 차이점을 파악한다
3. 차이점 목록을 정리한다: 새 엔드포인트, 변경된 필드, 삭제된 필드

### Step 3: API 코드 반영

차이점이 있으면 코드를 업데이트한다.

**새 엔드포인트 추가 시:**
- `{{monorepo.packagesDir}}/api/<도메인>/<버전>/` 하위에 URL pathname 구조와 1:1 매핑되는 폴더 생성 (배치 규칙은 `{{api.pathPattern}}`)
- `types.ts`: 스키마(`{{api.schemaLib}}`) 또는 TypeScript 타입으로 Request/Response 정의
- `index.ts`: client(`{{api.client}}`)를 사용한 API 함수 작성, `ApiResponse<T>` 제네릭으로 응답 래핑
- `{{monorepo.packagesDir}}/api/<도메인>/index.ts` (barrel export)에 새 모듈 export 추가

**기존 엔드포인트 변경 시:**
- `types.ts`의 필드를 추가/수정/삭제

> 정확한 코드 패턴은 동일 도메인 기존 모듈을 관찰해 따른다. 신규 모듈 생성은 `/new-api` 스킬과 동일한 규칙.

### Step 4: Postman Collection 폴더 확인

1. `getCollections`로 Collection 검색 (매핑 테이블 참조)
2. `getCollection`으로 폴더 구조 및 폴더 ID 확인
3. 사용자가 지정한 폴더 또는 매핑된 폴더에 API 추가

### Step 5: Postman에 API 추가

`createCollectionRequest`로 각 엔드포인트를 폴더에 추가한다.

**중요 규칙:**
- `folderId`: 반드시 대상 폴더 ID 지정
- **POST/PUT 요청은 반드시 request body 포함:**
  - `dataMode`: `"raw"`
  - `rawModeData`: JSON 문자열 (실제 스키마 기반 예시 값)
  - `dataOptions`: `{"raw": {"language": "json"}}`
  - `headerData`: `[{"key": "Content-Type", "value": "application/json"}]`
- **Path 변수**: URL의 `:param` 형태 사용 (예: `:goodsId`)
- **Base URL 변수**: 매핑 테이블의 변수 사용 (예: `{{postman.collections.<domain>.baseUrlVar}}/v1/...`)

**기존 요청 업데이트 시:**
- `updateCollectionRequest`로 `requestId` 지정하여 수정 (PATCH 동작)

### Step 6: 결과 보고

사용자에게 결과를 보고한다:
- API 코드 변경사항: 새 파일, 수정된 파일, barrel export 추가 여부
- Postman 변경사항: 추가/수정된 엔드포인트 수, 폴더 구조
- 변경 없음이면 "이미 최신 상태" 안내

## 주의사항

- Postman MCP 서버가 연결되어 있지 않으면 대안으로 JSON 파일을 프로젝트 루트에 저장한다
- WebFetch로 파싱한 API 문서는 불완전할 수 있으므로, 기존 코드와 큰 차이가 있으면 사용자에게 확인받는다
- API 폴더 구조는 URL pathname과 1:1 매핑 필수 (같은 path의 GET/POST/PUT/DELETE는 한 `index.ts`에 OK)
- workspace/collection ID, base URL 변수 등 회사 고유값은 `conventions.json`의 `postman` 섹션에서 읽고, 없으면 사용자에게 물어본다 (발명 금지)

## Fallback: JSON 파일 저장

Postman MCP를 사용할 수 없는 경우:
1. Postman Collection v2.1 JSON 형식으로 파일 생성
2. 프로젝트 루트에 `{collection-name}.postman_collection.json`으로 저장
3. 사용자에게 Postman에서 Import하도록 안내
