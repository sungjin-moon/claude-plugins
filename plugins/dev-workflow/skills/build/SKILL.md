---
name: build
description: 앱을 빌드하고 결과(에러, 경고, 번들 크기)를 분석합니다. "/build", "빌드해줘", "배포 전 빌드 확인", "빌드 에러 확인" 등의 요청에 사용하세요.
argument-hint: '[app-name]'
---

$ARGUMENTS 앱을 빌드하고 결과를 분석해줘.
이 스킬은 특정 프로젝트에 고정돼 있지 않다. **실행 시점에 프로젝트의 컨벤션을 먼저 읽고** 그대로 따른다.

## 0. 컨벤션 로드 (항상 먼저)

1. 프로젝트 루트에서 다음을 읽는다:
   - `ARCHITECTURE.md` — 빌드 파이프라인·스타일 시스템 등 서술 규칙(있으면)
   - `.claude/conventions.json` — `monorepo`, `apps` 노브
2. 아래에서 `{{...}}`는 컨벤션에서 읽은 값을 의미한다:
   - `{{monorepo.tool}}` = `monorepo.tool` (turbo | nx | none, 기본 `turbo`)
   - `{{monorepo.packageManager}}` = `monorepo.packageManager` (pnpm | npm | yarn, 기본 `pnpm`)
   - `{{apps}}` = `apps` 목록. 각 앱의 `name` / `tier`(new | legacy) / `styling`을 빌드 명령·에러 분류에 사용
3. `conventions.json`이 없으면 `package.json`(scripts·workspaces)에서 빌드 명령을 추론하고, **앱 식별이 애매하면 추측하지 말고 사용자에게 물어본다.**

> 아래 명령은 `{{monorepo.tool}} == turbo`, `{{monorepo.packageManager}} == pnpm` 가정의 예시다. 다른 도구면 그에 맞춰 치환한다(예: nx면 `nx build <app>`).

## 동작 규칙

### 인자가 없는 경우

전체 프로젝트 빌드 (패키지 매니저의 build 스크립트):

```bash
{{monorepo.packageManager}} build
```

### 앱 이름이 지정된 경우

먼저 `{{apps}}`에서 해당 앱의 `tier`를 확인한다.

**신규 앱 (`tier == new`)** — 의존 패키지까지 함께 빌드:

```bash
{{monorepo.packageManager}} {{monorepo.tool}} run build --filter=$ARGUMENTS...
```

> `--filter=<app>...`의 `...`은 의존 패키지(공용 UI/lib 등)까지 함께 빌드한다는 의미. 신규 앱은 보통 codegen(예: Panda CSS)이 필요해 이 형태가 권장된다.

**레거시 앱 (`tier == legacy`)**:

```bash
{{monorepo.packageManager}} --filter $ARGUMENTS build
```

> 앱별 `tier`를 모르면 `conventions.json`의 `apps`를 우선 확인하고, 없으면 사용자에게 신규/레거시 여부를 물어본다.

## 빌드 에러 발생 시

1. 에러 로그를 분석하여 원인 분류:

   - **타입 에러**: TypeScript 컴파일 에러
   - **린트 에러**: ESLint 규칙 위반 (빌드 전 lint가 먼저 실행되는 파이프라인이면)
   - **스타일 codegen 에러**: 앱 `styling` 시스템(Panda CSS 등)의 codegen 실패
   - **의존성 에러**: 패키지 누락, 버전 불일치
   - **빌드 설정 에러**: next.config, `{{monorepo.tool}}` 설정(turbo.json 등)

2. 에러별 수정 방안 제안

3. 수정할지 물어봐줘

## 빌드 성공 시

- 빌드 결과 요약 (소요 시간, 번들 크기 등 표시된 정보)
- 경고사항이 있으면 안내
