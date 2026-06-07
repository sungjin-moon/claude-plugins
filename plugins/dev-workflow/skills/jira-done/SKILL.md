---
name: jira-done
description: 수정 완료된 Jira 이슈에 AI 마커 댓글을 달고 완료 상태(예 In Review)로 전환한 뒤, 상위 Epic 보고자에게 Slack 알림을 전송합니다. "/jira-done PROJ-XXXX", "PROJ-XXXX QA 넘겨줘", "이슈 마무리", "끝났어 댓글 달고 상태 바꿔줘", "QA로 넘겨", "PROJ-XXXX 끝났어" 등 작업 완료 후 댓글 + 상태 전환 + Slack 알림이 필요한 요청에는 반드시 이 스킬을 사용하세요. 다중 키 지원. 코드 수정은 하지 않음(`/jira-fix` 책임).
argument-hint: '[issue-key ...]'
---

수정이 완료된 Jira 이슈에 AI 마커 댓글을 달고 상태를 **완료 상태**(`{{jira.doneStatus}}`)로 전환해줘.
이 스킬은 특정 프로젝트에 고정돼 있지 않다. **실행 시점에 프로젝트의 컨벤션을 먼저 읽고** 그대로 따른다.

대상: $ARGUMENTS

예시:
- `/jira-done PROJ-6628`
- `/jira-done PROJ-6628 PROJ-6629 PROJ-6630 PROJ-6631`
- `/jira-done PROJ-6628,PROJ-6629` (콤마 구분도 허용)

## 0. 컨벤션 로드 (항상 먼저)

1. 프로젝트 루트에서 다음을 읽는다:
   - `ARCHITECTURE.md` — 워크플로우 관련 서술 규칙(있으면)
   - `.claude/conventions.json` — `workflow.jira`, `workflow.slack` 노브
2. 아래에서 `{{...}}`는 컨벤션에서 읽은 값을 의미한다:
   - `{{jira.projectKeys}}` = `workflow.jira.projectKeys` (예: `["PROJ"]`)
   - `{{jira.doneStatus}}` = `workflow.jira.doneStatus` (완료 시 전환할 상태 이름, 예: `In Review`)
   - `{{jira.aiMarker}}` = `workflow.jira.aiMarker` (AI 처리 마커 문구, 예: `🤖 AI 처리됨`)
   - `{{slack.notifyChannel}}` = `workflow.slack.notifyChannel` (기본 알림 채널 ID/이름)
   - `{{slack.notifyEpicReporter}}` = `workflow.slack.notifyEpicReporter` (상위 Epic 보고자에게 멘션 여부)
3. `conventions.json`이 없으면: 완료 상태 이름·알림 채널을 **추측하지 말고 사용자에게 물어본다.** workspace 도메인은 사용자 환경/직전 컨텍스트에서 추출.

> 이 문서의 모든 `In Review` 표기는 `{{jira.doneStatus}}`를, 모든 채널 ID 표기는 `{{slack.notifyChannel}}`를 의미하는 placeholder다. 실제 값은 컨벤션에서 읽는다.

## 1) 입력 파싱

`$ARGUMENTS`에서 issue key 목록 추출.

- 공백 / 콤마 / 줄바꿈으로 구분된 다중 키
- 키 형식: `[A-Z][A-Z0-9_]+-\d+` (`{{jira.projectKeys}}`가 있으면 그 키들을 우선 매칭)
- 인자가 비어 있으면 직전 대화 컨텍스트에서 프로젝트 키 패턴(예: `PROJ-XXXX`)을 자동 탐색 후 사용자 확인

### 입력 모드 자동 감지

키들 각각의 `issuetype.name`을 step 2에서 확인한 뒤 두 모드 중 하나로 분기:

- **A) 하위 이슈 모드 (기본)** — 키가 `에픽`이 아닌 경우. 댓글 등록 + `{{jira.doneStatus}}` 전환 + Slack 알림 (step 3~8 전부 실행). 여러 키가 섞여 있어도 OK
- **B) Epic 모드** — 키가 `에픽` 타입인 경우. **댓글/전환은 스킵** (Epic 자체를 완료 상태로 전환하지 않음). 해당 Epic의 하위 중 현재 `{{jira.doneStatus}}` 상태인 이슈들을 자동 수집해서 Slack 알림만 전송 (step 8만 실행)

Epic 모드는 "이미 여러 티켓을 완료 상태로 넘겨놨는데 한 번에 보고자에게 알림만 다시 보내고 싶다" 같은 상황을 위함. 두 모드를 한 호출에 섞지 말 것 (혼란 방지).

## 2) 각 이슈별 컨텍스트 수집

각 key마다 1회씩:

```
mcp__mcp-atlassian__jira_get_issue(issue_key=<KEY>)
```

응답에서 다음 3개 필드 추출 (본문(description)은 불필요):

- `summary` — 작업 요약 prefill에 사용
- `reporter.display_name` — 결과 리포트 표의 "보고자" 컬럼
- `status.name` — 현재(이전) 상태. 결과 표에서 "이전상태 → {{jira.doneStatus}}" 화살표 표기에 사용

내부적으로 `{ key, summary, reporter, prevStatus }` 구조로 모아둠.

## 3) 댓글 본문 작성

각 이슈별로 한 줄 ~ 세 줄의 작업 요약을 직전 대화 컨텍스트에서 뽑아내거나, 사용자에게 짧게 확인. 그리고 다음 템플릿 적용:

```
{{jira.aiMarker}} _Claude Code(AI)가 작성한 댓글입니다._

<요청 사항 호응> + <어디서 어떻게 보이는지 사용자 관점 1~2문장>
<선택: 부수적 변화 한 줄>

QA 검토 부탁드립니다 🙏
```

> 첫 줄 마커는 `{{jira.aiMarker}}`로 시작해 AI 자동 작성임을 즉시 인지시킨다. `conventions.json`에 `aiMarker`가 없으면 사용자에게 마커 문구를 물어보거나 `🤖 AI 처리됨` 같은 기본 표식을 쓴다.

### 작성 규칙 (비개발자 친화 톤)

Jira 댓글은 QA·디자이너·PM 등 **비개발자가 읽는다**. 댓글 본문은 코드 변경의 기술적 디테일이 아니라, **사용자가 화면에서 보는 변화**로 설명한다. 개발자 톤(컴포넌트명·prop명·함수명·CSS 토큰)이 그대로 노출되면 읽는 사람이 의미 파악에 시간을 더 쓰게 된다.

1. **사용자 관점 우선** — "어디서 무엇이 어떻게 보이는지"로 기술. 구현 디테일(컴포넌트명·함수·prop·CSS prop명) 노출 금지
2. **백틱 자제** — 정말 외래어 약자(QR, URL)만. `fixedWeeks`, `: N개` 같은 코드 표기는 자연어로 번역
3. **장소는 사용자가 아는 이름** — `UserOrderListContainer` ❌ → "구매내역 카드" ✅. Figma·디자인 시안·기획서에 나오는 화면명 사용
4. **추상→구체** — "수량 표시" 보다 "'1개', '2개' 같은 표시" 식으로 예시 한 단어 넣기
5. **요청 호응** — "요청하신 대로", "말씀하신 부분", "이슈 내용대로" 식 짧은 한 마디로 시작 (보고자가 자기 요청이 반영됐다고 확신할 수 있게)
6. **1~3문장 짧게** — 사실 + 변화만. 자랑·과시 금지
7. **긍정 표현** — "삭제했습니다" 보다 "보이지 않도록 했습니다" / "막았습니다" 보다 "선택하기 전엔 누를 수 없도록 했습니다"
8. **멘션 안 넣음** — MCP가 accountId를 노출하지 않아 `@이름` / `[~이름]`은 텍스트로만 남고 알림이 가지 않음

### 예시 (Before → After)

**개발자 톤 (지양):**
> 옵션 선택 > 자세히 팝업에서 고정비 항목은 수량(`: N개`)을 숨기고 이름만 노출되도록 수정 완료했습니다.

**비개발자 친화 톤 (권장):**
> 요청하신 대로 옵션 선택 후 '자세히' 팝업에서 고정비 항목은 '1개', '2개' 같은 수량 표시 없이 이름만 보이도록 수정했습니다.

---

**개발자 톤:**
> 날짜 선택 캘린더에서 `fixedWeeks` 옵션을 제거하여, 월별로 필요한 주(4~6주)만 동적으로 렌더되도록 수정 완료했습니다.

**비개발자 친화 톤:**
> 날짜 선택 캘린더가 모든 달을 6줄로 채우던 동작을 바꿔서, 달마다 필요한 줄(4~6줄)만 표시하도록 했습니다. 5줄로 끝나는 달은 하단의 빈 공간이 사라집니다.

---

**개발자 톤:**
> 시간 선택 전에는 수량 +/- 버튼이 비활성화되도록 수정 완료했습니다.

**비개발자 친화 톤:**
> 시간을 먼저 선택하지 않으면 수량을 더하거나 뺄 수 없도록 막아두었습니다. 시간을 고르기 전엔 가격도 회색으로 흐리게 표시됩니다.

---

작성한 본문을 사용자에게 한 번 보여주고 일괄 확인 받기. "이대로 등록?" 식 짧은 확인.

## 4) 댓글 등록

각 이슈에:
```
mcp__mcp-atlassian__jira_add_comment(
  issue_key=<KEY>,
  body=<위 템플릿 본문>
)
```

응답에서 `id`(comment id) 보존.

## 5) 완료 상태 transition id 조회 (1회만)

**전체 이슈를 통틀어 한 번만 호출하고 모든 이슈에 재사용** — 같은 프로젝트의 같은 워크플로우면 transition id가 동일하기 때문. 매번 호출하면 N배의 불필요한 API 콜이 발생하므로 첫 번째 키로만 조회:

```
mcp__mcp-atlassian__jira_get_transitions(issue_key=<첫 번째 KEY>)
```

응답 리스트에서 `name === "{{jira.doneStatus}}"`인 항목의 `id` 추출 → `doneTransitionId`로 보존. 못 찾으면 사용자에게 가능한 transition 목록 보여주고 선택받기.

(예외적으로 한 번에 여러 프로젝트 키가 섞여 있으면(`PROJ-X SHOP-Y`) 프로젝트별로 한 번씩 — 보통은 단일 프로젝트라 1회로 충분.)

## 6) 상태 전환

각 이슈:
```
mcp__mcp-atlassian__jira_transition_issue(
  issue_key=<KEY>,
  transition_id=<위에서 찾은 id>
)
```

## 7) 결과 리포트

step 2에서 모은 `{ key, reporter, prevStatus }` 데이터로 표 작성:

```
| 이슈 | 보고자 | 댓글 | 상태 |
|---|---|---|---|
| [<KEY>](URL) | <reporter.display_name> | ✅ | <prevStatus> → **{{jira.doneStatus}}** |
| ... |
```

- `URL`: `https://<workspace>.atlassian.net/browse/<KEY>` 형식. workspace 도메인은 사용자 환경에서 추출하거나, 안 되면 키만 텍스트로 표기
- `prevStatus`: step 2에서 받은 실제 이전 상태(`해결중`, `작업완료` 등). **하드코딩 금지**
- 댓글 실패 시 `❌`, 상태 전환 실패 시 화살표 우측을 `(전환 실패)`로

마지막에 멘션 알림 한계 한 줄 부연:

> 참고: MCP가 accountId를 노출하지 않아 실제 멘션 알림은 안 갑니다. 보고자는 watcher라 상태 전환 자체로 알림은 발송됩니다.

## 8) Slack 알림 전송

> 이 단계는 `{{slack.notifyEpicReporter}}`가 true(또는 미설정 시 기본 동작)일 때 수행한다. `workflow.slack` 설정이 아예 없으면 Slack 단계를 건너뛰고 그 사실을 결과 표에 한 줄로 남긴다.

완료 상태로 넘긴 사실을 **상위 Epic의 보고자**에게 Slack으로 즉시 알린다. 보고자 입장에서 "내 요청이 어디까지 진행됐는지" 한눈에 보고 멘션 알림으로 인지할 수 있게 하는 것이 목적.

### 8-1) 알림 대상 Epic 식별

**A) 하위 이슈 모드** — step 2의 `jira_get_issue` 응답에서 각 이슈의 Epic key를 뽑는다. 보통 다음 위치 중 하나:
- `parent.key` (Jira Cloud 최신 — Story/Task가 Epic을 parent로 가짐, 응답의 `parent.fields.issuetype.name`이 `에픽`인지로 검증)
- `epic_link`, `customfield_10014` 등 (구버전/커스텀)

여러 이슈가 같은 Epic이면 한 번만 알림. Epic 없는 이슈는 결과 표에 "상위 Epic 없음"으로 표기하고 Slack 스킵.

**B) Epic 모드** — 입력 키 자체가 Epic이면 그 키가 곧 알림 대상. 별도 Epic 추출 불필요.

### 8-2) Epic 정보 + 완료 상태 하위 이슈 수집

각 Epic key에 대해:

1. Epic 메타 정보 조회 (한 번만):
```
mcp__mcp-atlassian__jira_get_issue(issue_key=<EPIC_KEY>)
```
추출: `summary` (메시지 헤더용), `reporter.display_name` (멘션 매칭용)

2. **현재 시점에 완료 상태(`{{jira.doneStatus}}`)인 하위 이슈** 자동 수집:
```
mcp__mcp-atlassian__jira_search(
  jql='parent = <EPIC_KEY> AND status = "{{jira.doneStatus}}"',
  fields="summary,status,reporter"
)
```

결과의 `key`, `summary`, `reporter.display_name`을 모은다. `key`와 `summary`는 메시지의 "반영된 이슈" 리스트, `reporter.display_name`은 8-3에서 추가 멘션 대상 식별에 사용 (Epic 보고자와 다른 보고자가 있다면 함께 멘션해야 하므로).

이번 호출로 새로 완료 전환한 이슈뿐 아니라 **기존에 이미 완료 상태였던 이슈까지 포함**되도록 JQL로 다시 조회하는 게 핵심 — 보고자는 Epic 전체 진척을 보고 싶지 한 번에 처리된 것만 보고 싶은 게 아니기 때문.

만약 결과가 0건이면:
- 하위 이슈 모드(A)인데 0건은 비정상 (방금 전환한 게 있어야 함). 그래도 발생했다면 사용자에게 보고하고 Slack 스킵
- **Epic 모드(B)에서 0건은 정상** — Epic을 처음 등록하고 아직 완료 상태로 넘어간 하위 이슈가 없는 경우. 이때는 메시지 톤을 "에픽 시작" 변형(8-4의 두 번째 템플릿)으로 바꿔 그대로 전송

### 8-3) Slack 사용자 매칭 (Epic 보고자 + 하위 보고자)

알림은 **Epic 보고자 + 하위 이슈에 Epic과 다른 보고자가 있다면 그들도 함께 멘션**해야 한다. 이유: 하위 이슈를 등록한 사람도 "내 요청이 처리됐다"는 사실을 직접 인지해야 함. Epic 보고자만 멘션하면 다른 보고자는 알림을 못 받음.

절차:

1. **멘션 대상 명단 작성**:
   - Epic 보고자 (`reporter.display_name`) → 무조건 포함
   - 8-2에서 수집한 완료 상태 하위 이슈들의 `reporter.display_name` 중 Epic 보고자와 다른 사람만 추가 → step 8-2 JQL 호출 시 `fields="summary,status,reporter"`로 한 번에 가져오면 효율적
   - **에픽 첫 공유(Epic 모드 + 하위 0건 = "에픽 시작 알림", 8-4 Template B)일 때는 컨벤션에 정의된 고정 멘션 대상이 있으면 항상 추가** — 신규 에픽 공유 시 디자인/UX 검토 등을 위한 고정 참여자. 이 고정 대상은 개인 실명을 하드코딩하지 말고 **역할 기반**으로 컨벤션에서 읽는다(예: `conventions.json`의 `workflow.slack`에 디자인 리뷰어 같은 역할이 정의돼 있으면 그 사람, 없으면 사용자에게 "신규 에픽 공유 시 추가로 멘션할 디자인/UX 담당이 있나요?"를 물어본다). 이름으로 `users_search` 매칭하고 **Slack ID는 절대 하드코딩 금지**
   - 중복 제거 (set으로 한 번 처리)

2. **각 이름을 Slack 사용자로 매칭**:
   ```
   mcp__slack__users_search(query=<display_name>)
   ```
   - 정확히 1명 → 해당 `UserID`로 `<@UserID>` 멘션
   - 여러 명 → `RealName`이 정확히 일치하는 사람 우선
   - 매칭 실패 → 멘션 없이 텍스트로 "(이름)"만 표기

3. **운영 채널 전송 전 사용자 확인**:
   - Epic 보고자 외 추가 멘션 대상이 있으면, 운영 채널 전송 전 사용자에게 "<이름A> → @handleA, <이름B> → @handleB 매칭 맞나요?" 식으로 명시적 확인 요청. 잘못된 사람을 공개 채널에서 멘션하면 영향이 큼
   - 본인 DM(예: 본인 Slack handle)으로 미리보기를 보낼 때는 추가 확인 없이 진행 가능 (본인만 보는 채널이라 영향 없음)

4. **메시지 작성 시 멘션 나열**:
   - 공백으로 구분해 한 줄에: `<@U_EPIC> <@U_SUB1> <@U_SUB2>`
   - Epic 보고자를 항상 첫 번째에 두어 우선순위 명시

이름이 한글이라도 검색 동작 (워크스페이스의 RealName이 한글이면).

### 8-4) 메시지 작성 + 전송

**채널 결정 규칙**:
- 기본값: `{{slack.notifyChannel}}` (컨벤션의 운영 알림 채널)
- 사용자가 `/jira-done` 호출 시 채널 이름(`#xxx`) 또는 채널 ID(`Cxxxxxx`)를 함께 명시하면 → 그 채널로 전송
- 채널 인자 추출 방법: `$ARGUMENTS` 또는 직전 대화에 `#채널명` / 채널 ID 패턴이 보이면 그것을 사용. 둘 다 없으면 기본 채널 사용
- 사용자가 명시적으로 다른 채널을 지정한 경우, 메시지 전송 직전 미리보기에 "→ <대상 채널>로 전송합니다" 한 줄 같이 표시해서 실수 방지

본문 템플릿 (Slack 네이티브 mrkdwn — 링크는 `<URL|텍스트>`, 멘션은 `<@USER_ID>`). `<workspace>`는 컨벤션/환경에서 읽은 Jira 도메인.

**A. 반영된 이슈가 1건 이상일 때 (배포 완료 알림)**:

```
🤖 AI 자동 알림

<@USER_ID> 안녕하세요, 아래 항목 배포가 완료되어 QA 검토 단계로 넘어갔습니다.

📌 에픽: <https://<workspace>.atlassian.net/browse/EPIC_KEY|EPIC_KEY · Epic summary>

✅ 반영된 이슈
• <https://<workspace>.atlassian.net/browse/SUB_KEY|SUB_KEY · sub summary>
• <https://<workspace>.atlassian.net/browse/SUB_KEY|SUB_KEY · sub summary>

확인 부탁드립니다 🙏
```

**B. 반영된 이슈가 0건일 때 (Epic 모드 + 새로 등록된 에픽 — 에픽 시작 알림)**:

```
🤖 AI 자동 알림

<@USER_ID> 안녕하세요, 아래 항목 개발이 완료되어 QA 요청드립니다.

📌 에픽: <https://<workspace>.atlassian.net/browse/EPIC_KEY|EPIC_KEY · Epic summary>

확인 부탁드립니다 🙏
```

→ 0건일 땐 "배포 완료" 톤이 어색하니 "작업 시작" 톤으로 전환. "반영된 이슈" 블록은 통째로 생략.

규칙:
- 첫 줄 `🤖 AI 자동 알림`은 받는 사람이 본문을 읽기 전에 자동 발송임을 즉시 인지하도록 하는 표식. 내부 앱이라 Slack의 "다음을 사용하여 보냄" 어트리뷰션이 표시되지 않으므로 본문에 직접 명시
- 멘션 매칭 실패 시 `<@USER_ID>` 자리에 `보고자: <이름>` 사용 (기울임/볼드 없이)
- "반영된 이슈" 리스트는 **8-2에서 JQL로 다시 조회한 완료 상태 하위 전체**. 이번 호출에 안 들어 있던 키라도 현재 완료 상태면 포함
- 하위 이슈는 Epic과 동일한 형식(`SUB_KEY · sub summary`)으로 키+제목을 한 링크로 묶어 표기 — 보고자가 클릭 안 해도 어떤 항목인지 한눈에 파악 가능
- Epic도 같은 형식(`EPIC_KEY · Epic summary`) — 일관된 시각 패턴

사용자에게 본문 미리보기 한 번 보여주고 "이대로 보낼게요" 짧은 확인. 확인되면:

```
mcp__slack__conversations_add_message(
  channel_id=<결정된 채널 ID 또는 이름>,
  content_type="text/plain",
  text=<위 본문>
)
```

**중요 — `content_type="text/plain"` 필수**. 기본값(`text/markdown`)을 쓰면 Slack mrkdwn 문법이 표준 마크다운으로 오인식되어:
- `*텍스트*`가 볼드가 아닌 **기울임**으로 렌더됨
- `<URL|텍스트>` 링크가 깨짐 (마크다운 파서가 `|`를 못 다룸)
- 단일 줄바꿈이 무시됨 (마크다운은 두 칸 공백 또는 빈 줄이 필요)

`text/plain`이면 위 mrkdwn이 Slack에 그대로 전달되어 의도대로 렌더된다.

**중요 — 링크 텍스트 내 특수문자 이스케이프 필수**. `<URL|텍스트>` 문법에서 텍스트 안의 `>`, `<`, `&`는 Slack이 링크 종료 기호 등으로 오인해 링크가 잘리거나 깨진다. HTML 엔티티로 치환해야 함:
- `>` → `&gt;`
- `<` → `&lt;`
- `&` → `&amp;`

이슈 제목엔 `상품 상세 > 옵션 선택`, `국내 숙소 > 썸네일` 같이 `>` 가 자주 등장하므로 **이슈 summary를 메시지에 넣기 전에 반드시 위 치환을 적용**할 것. Slack 렌더링 시엔 자동으로 원래 문자로 보여진다.

### 8-5) 결과 표 보강

step 7 표 아래에 Slack 발송 결과:

```
**Slack 알림**
- EPIC_KEY (보고자: <이름> @SlackName) → ✅ 전송됨 (반영된 이슈 N건)
- EPIC_KEY2 (보고자 매칭 실패: <이름>) → ⚠️ 텍스트로 발송됨
```

여러 Epic을 처리했다면 Epic마다 한 줄. Epic 없는 이슈는 별도 한 줄로 "상위 Epic 없음 — Slack 스킵: <KEY>" 표시. Epic 모드(B)에서 하위 0건이면 "에픽 시작 알림으로 발송: EPIC_KEY" 표시 (스킵 아님 — 정상 발송).

## 하지 말 것

- **코드 수정 절대 금지** — 이건 `/jira-fix` 또는 메인 대화의 책임
- 작업 요약을 추측해서 부풀리지 않기 (모르면 사용자에게 물어보기)
- transition id 하드코딩 (`121` 등)하지 않기 — 환경마다 다를 수 있어 매번 조회
- Jira 댓글에 멘션 시도 (`@이름` / `[~이름]`) 추가하지 않기 — 어차피 텍스트로만 남고 혼란만 줌
- **Slack 사용자 ID 하드코딩 금지** — 매번 `users_search`로 조회. 사람마다 다르고 워크스페이스 따라 다름
- **Slack 채널 ID·고정 멘션 대상을 실명/ID로 하드코딩 금지** — 컨벤션(`workflow.slack`)에서 읽거나 사용자에게 물어보기
- **Slack에 같은 Epic 알림을 중복 전송하지 않기** — 그룹화는 반드시 1 Epic = 1 메시지
- **Slack 메시지 본문에 코드/구현 디테일 노출 금지** — Jira 댓글과 같은 톤(사용자 관점 짧은 요약)
- **Epic 보고자만 멘션하지 않기** — 하위 이슈에 다른 보고자가 있으면 함께 멘션 (그들도 자기 요청이 처리됐다는 알림을 받아야 함)
- **링크 텍스트 안의 `>`/`<`/`&` 이스케이프 빠뜨리지 않기** — 안 하면 링크가 잘려서 잘못 표시됨

## 에러 처리

- 이슈 없음 / 권한 없음 → 해당 키만 건너뛰고 나머지 진행, 결과 표에 ❌ 표기
- "{{jira.doneStatus}}" transition 없음 → 사용자에게 사용 가능한 status 목록 보여주고 선택받기
- `jira_add_comment` 실패 → 사용자에게 알리고 해당 이슈는 상태 전환도 건너뜀
- **Epic 정보 못 찾음** → 해당 이슈는 Slack 단계 스킵, 결과 표에 "상위 Epic 없음" 표기 (상태 전환은 그대로 진행)
- **Slack 사용자 매칭 실패** → 텍스트로 "보고자: <이름>" 표기 후 전송 진행 (멘션 알림은 안 가지만 누가 받아야 할지 메시지에 명시)
- **Slack 전송 실패 (네트워크/권한)** → 사용자에게 에러 노출하고 결과 표에 ❌ 표기. Jira 단계는 이미 완료된 상태이므로 롤백하지 않음
