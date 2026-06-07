---
name: typecheck
description: TypeScript 타입 체크를 실행하고 에러를 분석합니다. "/typecheck", "타입 체크해줘", "타입 에러 확인", "tsc 돌려줘" 등의 요청에 사용하세요.
argument-hint: '[app-name]'
---

TypeScript 타입 체크를 실행해줘.
이 스킬은 특정 프로젝트에 고정돼 있지 않다. **실행 시점에 프로젝트의 컨벤션을 먼저 읽고** 그대로 따른다.

대상: $ARGUMENTS

## 0. 컨벤션 로드 (항상 먼저)

1. 프로젝트 루트에서 `.claude/conventions.json`을 읽는다. 아래 값을 사용한다:
   - `{{pm}}` = `monorepo.packageManager` (기본 `pnpm`)
   - `{{appsDir}}` = `monorepo.appsDir` (기본 `apps`)
   - 사용 가능한 앱 목록 = `apps[].name`
2. `conventions.json`이 없으면 `package.json`(workspaces)·lockfile로 추론한다:
   - 패키지 매니저: `pnpm-lock.yaml`→pnpm, `yarn.lock`→yarn, `package-lock.json`→npm
   - 앱 목록: workspaces 글롭(`apps/*` 등)에서 디렉토리명 수집
3. 애매하면 추측하지 말고 사용자에게 물어본다.

## 동작 규칙

1. 인자가 없으면 프로젝트 루트에서 전체 타입 체크 실행:

   ```bash
   {{pm}} exec tsc --noEmit
   ```

2. 앱 이름이 지정되면 해당 앱 디렉토리에서 타입 체크 실행:

   ```bash
   {{pm}} exec tsc --noEmit -p {{appsDir}}/$ARGUMENTS/tsconfig.json
   ```

3. 사용 가능한 앱은 컨벤션에서 로드한 `apps[].name` 목록을 따른다.

4. 에러가 발견되면:
   - 에러를 파일별로 그룹핑하여 정리
   - 각 에러의 원인과 수정 방안을 제안
   - 수정할지 물어봐줘
