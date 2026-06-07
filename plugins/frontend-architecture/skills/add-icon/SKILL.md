---
name: add-icon
description: Figma에서 아이콘/이모지 SVG를 추출하여 프로젝트의 공용 UI 패키지에 컴포넌트로 등록합니다. "/add-icon", "아이콘 추가해줘", "이 Figma 노드에서 아이콘 등록", "ic_xxx UI 패키지에 넣어줘" 요청에 사용하세요. **중요: 코드 작성 중 공용 UI 패키지에 존재하지 않는 새 아이콘이 필요한 상황(Figma 디자인에 아이콘이 포함된 경우)에도 반드시 이 스킬을 사용하세요. 인라인 SVG를 직접 작성하지 말고, 이 스킬로 공용 UI 패키지에 정식 등록한 뒤 import하세요.** 이미 등록된 아이콘 사용법 안내나 아이콘 목록 조회는 이 스킬 대상이 아닙니다.
argument-hint: '<figma-url>'
---

Figma 디자인에서 아이콘 또는 이모지를 추출하고 공용 UI 패키지에 컴포넌트로 등록해줘.
이 스킬은 특정 프로젝트에 고정돼 있지 않다. **실행 시점에 프로젝트의 컨벤션을 먼저 읽고** 그 경로·빌드 명령을 사용한다.

인자: $ARGUMENTS
형식: `<Figma 아이콘/이모지 노드 URL>`
예시:

- `/add-icon https://figma.com/design/xxx...?node-id=123-456`

## 0. 컨벤션 로드 (항상 먼저)

1. 프로젝트 루트에서 다음을 읽는다:
   - `ARCHITECTURE.md` — 아이콘/에셋 관리 규칙이 서술돼 있으면 1차 기준.
   - (폴백) 프로젝트에 `ARCHITECTURE.md`가 없으면 → 플러그인 번들 base `${CLAUDE_PLUGIN_ROOT}/ARCHITECTURE.base.md`를 기본 패턴으로 읽어 따른다. (/frontend-architecture:init 으로 프로젝트 ARCHITECTURE.md를 만들어두면 더 좋다.)
   - `.claude/conventions.json` — `icons`, `packages`, `monorepo` 노브.
2. `conventions.json`에 `icons`가 없으면 `{{packagesDir}}` 하위에서 아이콘 컴포넌트 폴더·빌드 스크립트(package.json `scripts`)를 직접 찾아 추론하고, 애매하면 **추측하지 말고 사용자에게 물어본다.**
3. 아래에서 `{{...}}`는 컨벤션에서 읽은 값을 의미한다:
   - `{{icons.package}}` = 아이콘이 등록되는 패키지 (예: `@org/ui`)
   - `{{icons.componentsDir}}` = 아이콘 컴포넌트 디렉토리 (예: `packages/ui/src/components/icon`)
   - `{{icons.buildCommand}}` = 아이콘 빌드 명령 (예: `pnpm --filter @org/ui build:icons`)
   - `{{packages.uiComponentsImport}}` = 컴포넌트 import 경로 (예: `@org/ui/components`)
   - `{{packagesDir}}` = `monorepo.packagesDir` (기본 `packages`)

> **경로/빌드 명령은 위 컨벤션 값을 사용한다.** 아래 본문의 구체 경로·명령은 예시이며, 실제로는 `{{icons.*}}`로 치환해 실행한다.
> 이모지를 아이콘과 분리 관리하는 프로젝트라면, `{{icons.componentsDir}}`·`{{icons.buildCommand}}`에 대응하는 **이모지 변형 경로/명령**을 같은 규칙으로 도출한다(예: `.../icon` ↔ `.../emoji`, `build:icons` ↔ `build:emojis`). 분리하지 않는 프로젝트면 아이콘 경로만 사용한다. 애매하면 사용자에게 확인.

## 아이콘 vs 이모지 분류 기준

| 구분 | 아이콘 (icon) | 이모지 (emoji) |
|------|--------------|---------------|
| 색상 | 단일 컬러 (`currentColor`) | 다중 컬러 (원본 유지) |
| SVG 저장 경로 | (아이콘 svg 디렉토리) | (이모지 svg 디렉토리) |
| 컴포넌트 경로 | `{{icons.componentsDir}}` | (이모지 변형 디렉토리) |
| 네이밍 | `icon_{name}.svg` → `IconXxx` | `emoji_{name}.svg` → `EmojiXxx` |
| 빌드 명령 | `{{icons.buildCommand}}` | (이모지 변형 빌드 명령) |
| 색상 치환 | O (`currentColor`로 치환) | X (원본 색상 유지) |

**판별 방법:** SVG 내 사용된 색상 수로 판단. stroke/fill 색상이 1가지면 아이콘, 2가지 이상이면 이모지.

## 작업 순서

### 1. 기존 컴포넌트 확인

아이콘/이모지가 이미 존재하는지 확인한다.

```bash
ls {{icons.componentsDir}}
# 이모지를 분리 관리하면 이모지 변형 디렉토리도 확인
```

이미 동일한 컴포넌트가 있으면 안내하고 종료한다.

**주의:** 아이콘 이름에 사이즈 접미사(예: `_14`, `_22`)가 붙어있으면 별개의 아이콘이다. 예를 들어 `ic_upload`와 `ic_upload_14`는 서로 다른 아이콘이므로, `IconUpload`가 이미 있더라도 `IconUpload14`는 새로 추가해야 한다.

### 2. Figma에서 SVG 추출

Figma MCP의 `get_design_context`를 사용하여 노드의 SVG 코드를 가져온다.

### 3. 아이콘/이모지 분류

추출한 SVG의 색상 수를 확인하여 아이콘 또는 이모지로 분류한다.

### 4. SVG 파일 저장

분류에 따라 해당 폴더에 저장한다(아이콘 svg 디렉토리 / 이모지 svg 디렉토리).

**네이밍 규칙:**
- Figma 이름이 `ic_{name}` 형식이면 → `icon_{name}.svg` 또는 `emoji_{name}.svg`로 변환
- 예 (아이콘): `ic_refresh_22` → `icon_refresh_22.svg`
- 예 (이모지): `ic_google` → `emoji_google.svg`
- 소문자 + snake_case 유지

### 5. 컴포넌트 빌드

**아이콘인 경우:**
```bash
{{icons.buildCommand}}
```
- 색상을 `currentColor`로 치환 (CSS color 제어 가능)
- `icon_search.svg` → `IconSearch.tsx`

**이모지인 경우:**
```bash
# {{icons.buildCommand}}의 이모지 변형 (예: build:icons → build:emojis)
```
- 원본 색상 유지 (다중 컬러)
- `emoji_google.svg` → `EmojiGoogle.tsx`

**주의:** 빌드 명령은 해당 컴포넌트 폴더(`{{icons.componentsDir}}` 또는 이모지 변형)를 통째로 삭제 후 재생성하므로, `index.tsx` barrel 파일도 삭제됨.

### 6. barrel export 갱신

빌드 후 `index.tsx`가 삭제되었으므로, 해당 폴더 내 모든 `.tsx` 파일(index.tsx 제외)을 기준으로 재생성한다.

**아이콘:** `{{icons.componentsDir}}/index.tsx`
```typescript
export { default as IconBack22 } from './IconBack22'
export { default as IconSearch } from './IconSearch'
// ... 모든 아이콘 나열
```

**이모지:** (이모지 변형 디렉토리)`/index.tsx`
```typescript
export { default as EmojiGoogle } from './EmojiGoogle'
export { default as EmojiTranslationEn } from './EmojiTranslationEn'
// ... 모든 이모지 나열
```

### 7. 결과 리포트

- 분류: 아이콘 또는 이모지
- 추가된 파일명
- 생성된 컴포넌트명
- import 방법: `import { IconXxx } from '{{packages.uiComponentsImport}}'` 또는 `import { EmojiXxx } from '{{packages.uiComponentsImport}}'`

## 주의사항

- Figma 로그인 필요 (MCP 권한)
- 빌드 명령은 해당 폴더 전체를 재생성하므로 수동 수정 사항은 유지되지 않음
- 아이콘 빌드 시 SVG 내 디자인 시스템 정의 색상은 자동으로 `currentColor`로 치환됨 (치환 대상 색상 목록은 프로젝트의 아이콘 빌드 스크립트 설정을 따른다)
- 이모지 빌드 시 색상 치환 없음 (원본 유지)
