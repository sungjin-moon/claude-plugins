---
name: design-review
description: Figma 디자인과 실제 구현 코드의 일치 여부를 검증하고 차이점을 리포트합니다. "/design-review", "디자인 검증", "피그마랑 코드 비교", "구현이 디자인 스펙대로 됐는지 확인" 요청에 사용하세요. 코드를 새로 만드는 게 아니라 기존 구현과 Figma의 차이를 분석할 때 사용합니다.
argument-hint: '<figma-url> <code-path> [local-url]'
---

Figma 디자인 명세와 실제 구현된 코드를 비교하여 일치도를 검증하고 차이점을 리포트해줘.
이 스킬은 특정 프로젝트에 고정돼 있지 않다. **실행 시점에 프로젝트의 컨벤션을 먼저 읽고** 그 프로젝트의 토큰·컴포넌트 규칙대로 검증한다.

인자: $ARGUMENTS
형식: `<Figma 링크> <코드 경로> [로컬 URL]`
예시:

- `/design-review https://figma.com/design/xxx... {{monorepo.appsDir}}/<app>/app/v2/orders/`
- `/design-review https://figma.com/design/xxx... {{monorepo.appsDir}}/<app>/app/v2/orders/ http://localhost:<port>/v2/orders`

## 0. 컨벤션 로드 (항상 먼저)

1. 프로젝트 루트에서 다음을 읽는다:
   - `ARCHITECTURE.md` — 디자인 토큰·시멘틱 토큰·컴포넌트·import 규칙의 **서술적 원천 (우선)**
   - (폴백) 프로젝트에 `ARCHITECTURE.md`가 없으면 → 플러그인 번들 base `${CLAUDE_PLUGIN_ROOT}/ARCHITECTURE.base.md`를 기본 패턴으로 읽어 따른다. (/frontend-architecture:init 으로 프로젝트 ARCHITECTURE.md를 만들어두면 더 좋다.)
   - `.claude/conventions.json` — `apps`, `packages`, `styling`, `icons`, `monorepo` 노브
2. `conventions.json`이 없으면 `package.json`(workspaces)·폴더 구조에서 추론하고, 애매하면 **추측하지 말고 사용자에게 물어본다.**
3. 아래에서 `{{...}}`는 컨벤션에서 읽은 값을 의미한다:
   - `{{monorepo.appsDir}}` = `monorepo.appsDir` (기본 `apps`)
   - `{{monorepo.packagesDir}}` = `monorepo.packagesDir` (기본 `packages`)
   - `{{app}}` = 코드 경로가 속한 앱의 `apps[]` 항목 (name, port, tier, styling 등)
   - `{{packages.ui}}` = `packages.ui` (공용 UI 패키지, 예: `@org/ui`)
   - `{{styling.system}}` = 신규 앱 스타일 시스템 (panda | css-modules | tailwind ...)
   - `{{styling.semanticTokens}}` = 시멘틱 토큰 강제 여부
   - `{{styling.tokenDocSection}}` = 토큰 규칙이 적힌 위치 (없으면 `ARCHITECTURE.md`)
   - `{{icons.componentsDir}}` = 아이콘 컴포넌트 디렉토리
4. 코드 경로(`<code-path>`)로 대상 앱을 식별하고, 그 앱의 `tier`/`styling`에 맞춰 아래 검증 기준을 적용한다.

> 시멘틱 토큰 매핑 규칙, 텍스트 스타일 규칙은 항상 `ARCHITECTURE.md`(또는 `styling.tokenDocSection`)를 우선 원천으로 삼는다. 아래 예시는 그 규칙을 설명하기 위한 참고일 뿐, 프로젝트 문서가 다르면 문서를 따른다.

## 검증 단계

### 1단계: 코드 기반 분석 (frontend-design-reviewer agent)

```
Task(
  subagent_type="frontend-design-reviewer",
  prompt="
    Figma URL: $ARGUMENTS에서 첫 번째 인자
    대상 코드 경로: $ARGUMENTS에서 두 번째 인자

    먼저 ARCHITECTURE.md + .claude/conventions.json을 읽고,
    위 Figma 디자인과 코드를 비교하여 그 프로젝트 컨벤션 기준으로 일치도를 검증하고 상세 리포트를 작성하세요.
  "
)
```

### 2단계: 시각적 비교 (Playwright MCP)

로컬 URL이 제공된 경우, Playwright MCP로 실제 렌더링 결과를 Figma 스크린샷과 비교합니다.

#### 워크플로우

1. **Figma 스크린샷 가져오기**
   - `mcp__figma-remote-mcp__get_screenshot`으로 디자인 스크린샷 획득

2. **브라우저에서 실제 화면 캡처**
   - `mcp__playwright__browser_navigate`로 로컬 URL 접속 (포트는 대상 앱의 `{{app.port}}`)
   - 필요시 `mcp__playwright__browser_click` 등으로 특정 상태 재현 (모달 열기, 탭 전환 등)
   - `mcp__playwright__browser_take_screenshot`으로 스크린샷 촬영

3. **두 이미지 비교**
   - Figma 스크린샷과 브라우저 스크린샷을 나란히 비교
   - 레이아웃, 간격, 색상, 폰트 등 시각적 차이점 식별

4. **브라우저 닫기**
   - `mcp__playwright__browser_close`로 브라우저 종료

#### 뷰포트 사이즈

- **PC**: width 1920
- **Mobile**: width 360

`mcp__playwright__browser_resize`로 캡처 전 너비를 설정하고, `browser_take_screenshot`의 `fullPage: true`로 전체 페이지를 캡처한다. 높이는 임의값(예: 1080)으로 설정하되, fullPage 캡처이므로 실제 높이는 무관하다.

#### 주의사항

- 개발 서버가 실행 중이어야 함
- 모달, 드롭다운 등 인터랙션이 필요한 경우 사용자에게 조작 순서를 확인
- Mock 데이터와 디자인 예시 데이터의 차이는 무시

### 3단계: 종합 리포트

1단계(코드 분석)와 2단계(시각적 비교) 결과를 합쳐서 최종 리포트를 작성합니다.

## frontend-design-reviewer가 검증하는 항목

> 토큰/텍스트 스타일 매핑의 정확한 명칭과 규칙은 `ARCHITECTURE.md`(또는 `{{styling.tokenDocSection}}`)에서 읽는다. 아래 예시 명칭은 프로젝트마다 다를 수 있다.

### ⭐⭐⭐⭐⭐ 높은 정확도

1. **시멘틱 토큰 사용 검증** (`{{styling.semanticTokens}}`가 true인 경우 강제)

   - **신규 앱 (`{{styling.system}}`, 예: Panda CSS)**: Figma 색상 → 스타일 시스템 시멘틱 토큰 매핑 확인 (예: `var(--bg/secondary)` → `Bg.Secondary`)
   - **레거시 앱 (`{{app.styling}}`, 예: CSS Modules + SCSS)**: Figma 색상 → 해당 앱 토큰 함수의 시멘틱 토큰 매핑 확인 (예: `var(--bg/secondary)` → `token('Bg-Secondary')`)
   - 베이스 토큰 사용 감지: Figma에 시멘틱 변수가 있는데 `Gray-50`, `Pink-50` 등 베이스 토큰을 사용한 경우 지적
   - 하드코딩된 hex 값 감지
   - Spacing/Radius 토큰 사용 여부 확인 (예: `12px` 대신 토큰 사용)

2. **텍스트 스타일 검증**
   - **신규 앱**: Figma 텍스트 스타일 → 스타일 시스템 textStyle 토큰 매핑 확인
   - **레거시 앱**: Figma 텍스트 스타일 → 해당 앱의 mixin/스타일 매핑 확인 (예: `Body_3_SB`, `Body_2_SB`)
   - 하드코딩된 fontSize/fontWeight 감지

### ⭐⭐⭐⭐ 중상 정확도

3. **{{packages.ui}} 컴포넌트 사용 검증**

   - Figma 디자인 요소 → `{{packages.ui}}` 컴포넌트 매핑 확인
   - 버튼 계열(예: BoxButton/TextButton) 올바른 사용 검증
   - 필드 계열(예: Field/DateRangeField/DateSingleField) 올바른 사용 검증
   - Tag 컴포넌트 사용 검증 (상태/카테고리 표시에 tone 기반 색상 분류)

   > 위 컴포넌트 명칭은 예시다. 실제 사용 가능한 컴포넌트는 `{{packages.ui}}`의 export와 `ARCHITECTURE.md`를 기준으로 한다.

4. **아이콘/이모지 검증**
   - Figma에서 `ic_*` 형태의 아이콘이 코드에서 인라인 SVG로 작성되었는지 감지 → `{{packages.ui}}` 아이콘 컴포넌트 사용 필수
   - Figma 디자인의 아이콘 노드(`ic_*`, `emoji_*`)를 개별 조회하여 코드에서 사용 중인 아이콘과 형태 비교
   - `{{icons.componentsDir}}`에 등록되지 않은 아이콘이 필요한 경우 `/add-icon`으로 추가 권장
   - `ic_*` 형태가 아닌 장식적 SVG(일러스트, 배경 등)는 인라인 SVG 허용

5. **요소 누락 검증**
   - Figma 디자인의 모든 요소가 코드에 구현되었는지 확인

### ⭐⭐⭐ 중간 정확도

5. **레이아웃 구조 검증**
   - Figma 레이아웃 구조 (flex, grid) 확인
   - 코드의 레이아웃 구조와 비교

### ⭐⭐ 중하 정확도 (코드만) → ⭐⭐⭐⭐ (Playwright 포함 시)

6. **간격/크기 검증**
   - 코드만: Figma 간격/크기 대략적 비교 (±10% 허용)
   - Playwright 포함: 실제 렌더링 결과와 직접 비교

## 리포트 형식

```markdown
# 디자인 리뷰 결과

## 요약

- 검증 대상: {{monorepo.appsDir}}/<app>/app/v2/admin/settings/
- 전체 일치도: ⭐⭐⭐⭐ (4/5)
- 검증 방법: 코드 분석 + 시각적 비교

## 일치 항목

- ...

## 차이 항목 (시각적 비교에서 발견)

| 항목 | Figma | 실제 렌더링 | 원인 |
|------|-------|------------|------|
| 모달 패딩 | 30px | 60px | 이중 패딩 (Modal + body) |

## 문제 항목

- ...

## 개선 제안

- ...
```

## 검증 범위

### 코드 분석으로 검증 가능

- 색상 토큰 사용 여부
- 텍스트 스타일 사용 여부
- `{{packages.ui}}` 컴포넌트 사용 여부
- 아이콘/이모지 컴포넌트 사용 여부 (인라인 SVG 감지)
- 레이아웃 구조 (대략)
- 요소 누락

### Figma API 접근 실패 시

- 리포트에 **"⚠️ Figma API 접근 실패 - 시각적 비교 불가"** 경고를 명시
- 코드 정적 분석만으로는 검증 불가한 항목을 별도 섹션으로 분리:
  - 아이콘/이모지 형태 일치 여부
  - 실제 간격/크기 정확도
  - 색상 토큰 매핑의 정확성
- 사용자에게 Figma 접근 복구 후 재검증을 권장

### Playwright 추가 시 검증 가능

- **실제 렌더링 결과**: 브라우저에서 보이는 최종 결과
- **간격/패딩**: CSS 상속/캐스케이드 효과 포함
- **컴포넌트 내부 스타일 충돌**: 부모 컴포넌트의 기본 스타일과의 충돌

### 검증 불가능

- **반응형**: 다양한 브레이크포인트 동작 (수동 확인 필요)
- **애니메이션**: 인터랙션 효과

## 사용 예시

```bash
# 코드 분석만 (로컬 URL 없음)
/design-review https://figma.com/design/abc123?node-id=1-2 {{monorepo.appsDir}}/<app>/app/v2/orders/

# 코드 분석 + 시각적 비교 (로컬 URL 포함)
/design-review https://figma.com/design/abc123?node-id=1-2 {{monorepo.appsDir}}/<app>/app/v2/orders/ http://localhost:<port>/v2/orders
```

## 권장 워크플로우

```bash
# 1. UI 생성
/ui-build https://figma.com/... <app> commission-policies

# 2. 개발 서버 실행
/dev <app>

# 3. 디자인 검증 (코드 + 시각적 비교)
/design-review https://figma.com/... {{monorepo.appsDir}}/<app>/app/v2/admin/settings/policies/ http://localhost:<port>/v2/admin/settings/policies

# 4. 아키텍처 검증
/review
```
