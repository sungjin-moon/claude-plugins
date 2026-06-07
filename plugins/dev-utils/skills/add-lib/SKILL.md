---
name: add-lib
description: 외부 라이브러리를 공용 lib 패키지에 추가하여 전체 앱에서 사용할 수 있도록 등록합니다. "/add-lib", "lib 패키지에 추가해줘", "라이브러리 등록", "새 lib 추가" 요청에 사용하세요. 이미 lib 패키지에 있는 패키지를 import하는 방법은 이 스킬 대상이 아닙니다.
argument-hint: "<library>"
---

공용 lib 패키지에 새 외부 라이브러리를 추가해줘.
이 스킬은 특정 프로젝트에 고정돼 있지 않다. **실행 시점에 프로젝트의 컨벤션을 먼저 읽고** 그대로 따른다.

인자: $ARGUMENTS
형식: `<라이브러리명>`
예시:
- `/add-lib date-fns`
- `/add-lib @tanstack/react-table`

## 0. 컨벤션 로드 (항상 먼저)

1. 프로젝트 루트에서 `.claude/conventions.json`을 읽는다. 아래 값을 사용한다:
   - `{{pm}}` = `monorepo.packageManager` (기본 `pnpm`)
   - `{{packagesDir}}` = `monorepo.packagesDir` (기본 `packages`)
   - `{{lib}}` = `packages.lib` — 외부 라이브러리 래퍼 패키지 이름 (예: `@org/lib`)
2. `conventions.json`이 없거나 `packages.lib`이 비어 있으면 추론한다:
   - 패키지 매니저: lockfile로 판별(`pnpm-lock.yaml`→pnpm, `yarn.lock`→yarn, `package-lock.json`→npm)
   - lib 패키지: `{{packagesDir}}/*/package.json`에서 외부 라이브러리를 re-export하는 패키지(보통 `*/lib` 또는 이름에 `lib` 포함)를 찾는다
3. lib 패키지를 확실히 특정할 수 없으면 **추측하지 말고 사용자에게 물어본다.**

## lib 패키지 위치

패키지 경로: `{{packagesDir}}/<lib 패키지 디렉토리>/`
(예: `{{lib}}`이 `@org/lib`이면 보통 `{{packagesDir}}/lib/`. 실제 경로는 워크스페이스 설정/디렉토리 구조로 확인한다.)

## 작업 순서

1. **기존 export 패턴 확인**
   - lib 패키지의 `package.json` exports 필드 구조 확인
   - 기존 라이브러리들이 어떻게 re-export되는지 패턴 파악

2. **라이브러리 설치**
   ```bash
   {{pm}} --filter {{lib}} add <라이브러리명>
   ```

3. **Re-export 파일 생성**
   - lib 패키지 디렉토리 안에 `<라이브러리명>/index.ts` 생성
   - 기존 패턴에 맞게 re-export 작성

4. **package.json exports 추가**
   - lib 패키지 `package.json`의 exports 필드에 새 entry 추가
   - 기존 패턴과 동일한 형식으로

5. **빌드 설정 확인**
   - 번들러 설정(tsup 등)에 entry point 추가 필요 여부 확인

## 완료 후

1. 추가된 라이브러리의 import 경로 안내
   ```typescript
   import { something } from '{{lib}}/<라이브러리명>'
   ```
2. 빌드 테스트 (`{{pm}} --filter {{lib}} build`)
3. 사용 예시 제공
