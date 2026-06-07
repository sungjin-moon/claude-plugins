---
name: new-component
description: 프로젝트 아키텍처 컨벤션에 맞는 React 컴포넌트를 생성합니다. "/new-component", "컴포넌트 만들어줘", "새 컴포넌트 생성", "xxx 컴포넌트 추가해줘" 요청에 사용하세요. 기존 컴포넌트 수정이 아닌 신규 생성에 사용합니다.
argument-hint: '<app> <name> [path]'
---

대상 프로젝트의 계층별 아키텍처 컨벤션에 따라 새 컴포넌트를 생성해줘.
이 스킬은 특정 프로젝트에 고정돼 있지 않다. **실행 시점에 프로젝트의 컨벤션을 먼저 읽고** 그대로 따른다.

인자: $ARGUMENTS
형식: `<앱이름> <컴포넌트명> [경로]`
예시:

- `/new-component admin ProductCard v2/products` → 특정 계층 전용
- `/new-component admin Form` → 앱 전역

## 0. 컨벤션 로드 (항상 먼저)

1. 프로젝트 루트에서 다음을 읽는다:
   - `ARCHITECTURE.md` — 계층 구조·import 규칙·네이밍의 서술적 원천
   - (폴백) 프로젝트에 `ARCHITECTURE.md`가 없으면 → 플러그인 번들 base `${CLAUDE_PLUGIN_ROOT}/ARCHITECTURE.base.md`를 기본 패턴으로 읽어 따른다. (/frontend-architecture:init 으로 프로젝트 ARCHITECTURE.md를 만들어두면 더 좋다.)
   - `.claude/conventions.json` — `apps`, `packages`, `styling`, `monorepo` 노브
2. `conventions.json`이 없으면 `package.json`(workspaces)·폴더 구조에서 추론하고, 애매하면 **추측하지 말고 사용자에게 물어본다.**
3. 아래에서 `{{...}}`는 컨벤션에서 읽은 값을 의미한다:
   - `{{appsDir}}` = `monorepo.appsDir` (기본 `apps`)
   - `{{app.tier}}` = 대상 앱의 `tier` (new | legacy)
   - `{{app.alias}}` / `{{app.styledSystemImport}}` = 앱 로컬 스타일 import
   - `{{ui.componentsImport}}` = `packages.uiComponentsImport` (공용 UI 컴포넌트)

## 배치 규칙

1. 경로가 지정된 경우 → `{{appsDir}}/<앱이름>/app/<경로>/components/<컴포넌트명>.tsx`
2. 경로가 없는 경우 → `{{appsDir}}/<앱이름>/ui/components/<컴포넌트명>.tsx` (앱 전역)

> 정확한 계층 폴더 규칙은 프로젝트 `ARCHITECTURE.md`를 우선한다. 위는 기본 가정.

## 사전 확인

1. 대상 앱의 기존 컴포넌트 패턴을 먼저 확인한다.
2. 같은 계층에 이미 존재하는 컴포넌트의 import 스타일을 파악한다.
3. **관찰한 패턴이 이 문서의 가정과 다르면, 관찰한 패턴을 따른다.**

## 컴포넌트 작성 규칙 — 신규(new) 앱 (`{{app.tier}} == new`)

### 스타일 시스템 import

`conventions.json`의 `app.styledSystemImport`를 사용한다. 예: 앱 로컬 styled-system을 쓰는 경우

```typescript
import { css, cx, cva } from '{{app.styledSystemImport}}'
```

> 공용 패키지의 styled-system이 아니라 각 앱 로컬 styled-system을 쓰는 프로젝트가 많다.
> 앱의 `panda.config.ts`(또는 동등한 설정)의 outdir로 확인.
> Panda가 아닌 스타일 시스템(css-modules, tailwind 등)이면 `app.styling`에 맞춰 작성.

### 공용 UI 컴포넌트 import

```typescript
import { Text, Button, TextInput } from '{{ui.componentsImport}}'
```

### 앱 내부 컴포넌트 import

```typescript
import { Translator } from '{{app.alias}}/ui/components'
```

### 네이밍

- 파일명: PascalCase (`ProductCard.tsx`)
- 앱 전역 컴포넌트: 접두사 없이 기능/역할 기반 (`Form.tsx`, `Table.tsx`)
- `'use client'`는 클라이언트 기능(useState, useEffect, 이벤트 핸들러 등)이 필요할 때만 추가

## 컴포넌트 작성 규칙 — 레거시(legacy) 앱 (`{{app.tier}} == legacy`)

- 기존 앱의 스타일 패턴 유지 (`app.styling` 참고)
- 해당 앱이 쓰던 패키지/규칙 그대로 사용
- 신규 패턴을 억지로 끌어오지 않는다 (호환성 우선)

## 동작

1. 컨벤션 로드 + 대상 앱의 기존 컴포넌트 패턴 확인
2. 컴포넌트 파일 생성
3. 동일 계층의 `index.ts`에 export 추가 (있는 경우)
4. 어떤 위치에 왜 배치했는지 설명
5. 사용 예시 코드 제공
