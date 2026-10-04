# 📙 3장. 에이전트 스킬

> **기준**: 「클로드코드로 시작하는 실전 에이전틱 코딩」 CHAPTER 03 목차 + Claude Code 공식 문서([code.claude.com/docs](https://code.claude.com/docs), 2026-10-04 확인). 책과 달라진 부분은 **📌 현재 공식 문서 기준**, 공식 문서에 전용 설명이 없는 부분은 **🔎 공식 문서 원칙에서 정리** 로 표시했다.
>
> 관련 노트: [1장](./01-getting-started.md) (컨텍스트 윈도우) · [2장](./02-workflow-and-settings.md) (권한 규칙) · [4장](./04-subagents.md) (서브에이전트)

---

## 3-1. 클로드 스킬의 개념과 구조

**스킬이란** — 지시문·참고 자료·작업 절차를 `SKILL.md` 파일 하나(와 보조 파일)로 묶은 것. Claude가 관련 있을 때 스스로 쓰거나, 사용자가 `/스킬이름` 으로 직접 호출한다.

**언제 만드나 (공식 문서)** — 같은 지시·체크리스트·절차를 채팅에 반복해서 붙여넣고 있을 때, `CLAUDE.md` 의 한 부분이 "사실"이 아니라 "절차"로 커졌을 때.

> 📌 **현재 공식 문서 기준**: Claude Code 스킬은 여러 AI 도구에서 쓰는 **Agent Skills 공개 표준**([agentskills.io](https://agentskills.io))을 따르고, 호출 제어·서브에이전트 실행·동적 컨텍스트 주입 같은 기능을 더했다. 예전 `.claude/commands/` 명령 파일도 계속 동작하지만 **스킬로 통합**되어 스킬이 권장된다 (이름이 같으면 스킬이 우선).

### 컨텍스트 윈도우는 공용 자산이다

**① 개념** — 대화, `CLAUDE.md`, 읽은 파일, 스킬이 모두 같은 컨텍스트 윈도우를 나눠 쓴다. 스킬을 많이 만들수록 그만큼 자리를 차지한다.

**② 왜 중요한가** — 공식 문서: 기능을 많이 붙이면 컨텍스트가 차는 것뿐 아니라 **잡음이 늘어** 스킬이 제대로 호출되지 않거나 규칙을 놓칠 수 있다. 또 한 번 로드된 스킬 본문은 이후 턴에도 계속 남으므로 "모든 줄이 반복되는 토큰 비용"이다.

**③ 그래서 스킬은 "필요할 때만" 로드되게 설계됐다** — 아래 "세 단계 점진적 작동 방식" 참고. 컨텍스트 기본 개념은 [1장](./01-getting-started.md).

**④ 기능별 로딩 비교 (공식 문서)**

| 기능 | 언제 로드 | 컨텍스트 비용 |
|---|---|---|
| `CLAUDE.md` | 세션 시작 (전체) | 매 요청 |
| 스킬 | 시작 시 설명만, 사용할 때 본문 | 낮음 (설명만 매 요청) |
| 서브에이전트 | 호출될 때 | 메인 세션과 분리 |
| hook | 이벤트 발생 시 | 0 (출력을 돌려줄 때만) |

🔗 [Understand context costs](https://code.claude.com/docs/en/features-overview#understand-context-costs)

### 스킬의 유형

**① 내용에 따른 유형 (공식 분류)**

| 유형 | 내용 | 예 |
|---|---|---|
| 참조형 (reference) | 지금 작업에 적용할 지식: 규칙, 패턴, 스타일 가이드, 도메인 지식. 대화 안에서 함께 쓰인다 | 사내 Java 코딩 규칙, MyBatis 작성 규칙 |
| 작업형 (task) | 특정 행동의 단계별 절차. 보통 `/이름` 으로 직접 호출하며 `disable-model-invocation: true` 를 붙인다 | 배포, 커밋, 릴리스 체크리스트 |

**② 저장 위치에 따른 유형**

| 위치 | 경로 | 적용 범위 |
|---|---|---|
| 개인 | `~/.claude/skills/이름/SKILL.md` | 내 PC의 모든 프로젝트 |
| 프로젝트 | `.claude/skills/이름/SKILL.md` | 이 저장소 (커밋해서 팀 공유) |
| 하위 폴더 | `하위폴더/.claude/skills/이름/SKILL.md` | 그 폴더 아래에서 시작한 세션 |
| 엔터프라이즈 | managed 설정 디렉터리 | 조직 전체 |
| 플러그인 | `플러그인/skills/이름/SKILL.md` | 플러그인이 켜진 곳, `/플러그인명:이름` 으로 호출 |
| 내장 (bundled) | Claude Code에 포함 | `/code-review`, `/debug`, `/batch`, `/loop`, `/run`, `/verify`, `/doctor` 등 |

🔗 [Skills](https://code.claude.com/docs/en/skills)

### SKILL.md 파일 구조

**① 구조** — 폴더 하나가 스킬 하나다. `SKILL.md` 는 맨 위 `---` 사이의 YAML frontmatter와 그 아래 마크다운 지시문으로 이뤄진다. (첫 줄이 `---` 가 아니면 frontmatter로 인식하지 않는다.)

```text
.claude/skills/review-jsp/
├── SKILL.md          (필수: 개요와 지시)
├── checklist.md      (보조: 상세 체크리스트)
└── scripts/
    └── find-scriptlet.ps1
```

**② 자주 쓰는 frontmatter 필드**

| 필드 | 의미 |
|---|---|
| `name` | `/` 명령 이름 (없으면 폴더명) |
| `description` | 무엇을 하고 **언제 쓰는지**. Claude가 자동 호출 여부를 판단하는 기준 (권장) |
| `when_to_use` | 호출 조건·예시 문장 추가 (description 뒤에 붙음) |
| `argument-hint`, `arguments` | 인자 힌트, 이름 있는 인자 (`$이름` 으로 치환) |
| `disable-model-invocation` | `true` 면 Claude가 스스로 호출하지 못하고 사용자만 `/이름` 으로 호출 |
| `user-invocable` | `false` 면 `/` 메뉴에서 숨기고 Claude만 사용 (배경 지식용) |
| `allowed-tools` / `disallowed-tools` | 그 턴에 승인 없이 쓸 도구 / 스킬이 활성인 동안 빼버릴 도구 |
| `paths` | 이 경로의 파일을 다룰 때만 자동 로드 (glob) |
| `model`, `effort` | 스킬이 활성인 동안 쓸 모델·노력 수준 |
| `context`, `agent`, `background` | `context: fork` 로 서브에이전트에서 실행 (3-3 마지막 절) |
| `shell` | `!` 명령 주입에 쓸 셸: `bash` 또는 `powershell` |
| `hooks` | 스킬이 호출될 때 등록되는 hook |
| `metadata` | 내 도구용 자유 형식 데이터 (Claude Code는 무시) |

**③ 본문에서 쓸 수 있는 치환값**

- `$ARGUMENTS` (전체 인자), `$0` / `$1` 또는 `$ARGUMENTS[0]` (순서별 인자), `$이름` (`arguments` 로 정의한 인자)
- `${CLAUDE_SKILL_DIR}` (스킬 폴더 경로), `${CLAUDE_PROJECT_DIR}` (프로젝트 루트), `${CLAUDE_SESSION_ID}`

**④ 예시** — 아래 "커스텀 스킬 개발" 참고.

🔗 [Skills – frontmatter reference](https://code.claude.com/docs/en/skills)

### 세 단계 점진적 작동 방식

**① 개념 (progressive disclosure)**

| 단계 | 언제 | 무엇이 로드되나 |
|---|---|---|
| 1. 설명 | 세션 시작 시 항상 | 이름 + description → Claude가 "어떤 스킬이 있는지" 안다 |
| 2. 본문 | 사용자가 `/이름` 으로 호출하거나 Claude가 관련 있다고 판단할 때 | `SKILL.md` 전체 본문 |
| 3. 보조 파일 | 본문의 지시에 따라 Claude가 필요할 때 읽음 | `reference.md`, 예시, 스크립트 결과 등 |

**② 왜 이렇게 하나** — 공식 문서: "긴 참고 자료도 필요해지기 전까지는 비용이 거의 없다."

**③ 실무 의미** — 수백 줄짜리 사내 코딩 규칙도 보조 파일로 두면, 평소에는 설명 한 줄만 컨텍스트를 차지한다.

🔗 [Skills](https://code.claude.com/docs/en/skills)

### 스킬 컨텍스트 비용

**① 비용이 생기는 지점**

- **설명**: 자동 호출 가능한 스킬은 이름·설명이 매 요청에 들어간다. `description` 과 `when_to_use` 를 합쳐 **1,536자**에서 잘린다 → 핵심 용도를 맨 앞에 쓴다.
- **본문**: 호출되면 대화에 들어가 이후 턴에도 남는다. 같은 내용으로 다시 호출하면 전체를 반복하지 않고 짧은 메모만 붙는다.
- **압축 후**: 호출된 스킬은 스킬당 앞부분 5,000토큰, 합계 25,000토큰까지만 다시 주입되고 오래된 것부터 빠진다 → 중요한 지시는 `SKILL.md` 앞쪽에 둔다.

**② 비용을 줄이는 방법**

| 방법 | 효과 |
|---|---|
| `disable-model-invocation: true` | 설명조차 컨텍스트에 넣지 않음. 호출할 때만 로드 (배포·커밋 같은 작업형) |
| `skillOverrides` 설정 | 남이 만든 스킬을 수정하지 않고 `on` / `name-only` / `user-invocable-only` / `off` 로 노출 조절 |
| `/skills` 메뉴 | 스킬을 골라 `Space` 로 위 상태를 바꾸고 `Esc` 로 저장 (`.claude/settings.local.json`) |
| 본문은 짧게, 상세는 보조 파일로 | 공식 문서: `SKILL.md` 는 **500줄 이하**, "어떻게·왜"를 설명하지 말고 "무엇을 할지"를 쓴다 |

```json
{
  "skillOverrides": {
    "legacy-context": "name-only",
    "deploy": "off"
  }
}
```

🔗 [Context cost by feature](https://code.claude.com/docs/en/features-overview#context-cost-by-feature) · [What survives compaction](https://code.claude.com/docs/en/context-window#what-survives-compaction)

### 도구 접근 권한 제한

> 📌 **현재 공식 문서 기준 – `allowed-tools` 는 도구를 "제한"하지 않는다.** 스킬을 호출한 **그 턴 동안** 나열한 도구를 승인 없이 쓰게 해주는 **사전 승인**이다. 다음 메시지를 보내면 해제되고, 나열하지 않은 도구도 그대로 쓸 수 있다(일반 권한 설정을 따름).

**① 도구를 다루는 방법 정리**

| 하고 싶은 것 | 방법 |
|---|---|
| 스킬 실행 중 확인 창 줄이기 | `allowed-tools: Bash(git add *) Bash(git commit *)` |
| 스킬이 활성인 동안 특정 도구 빼기 | `disallowed-tools: AskUserQuestion` |
| Claude가 쓸 수 있는 스킬 제한 | 권한 규칙 `Skill(commit)` (정확히), `Skill(deploy *)` (접두), 도구 전체는 `Skill` 을 deny |
| 도구가 정말 제한된 환경에서 실행 | `context: fork` 와 도구를 제한한 커스텀 서브에이전트를 함께 사용 (`tools` 필드, 4장) |

```yaml
---
name: commit
description: Stage and commit the current changes
disable-model-invocation: true
allowed-tools: Bash(git add *) Bash(git commit *) Bash(git status *)
---
```

> ⚠️ **보안 주의 (공식 문서)**: 프로젝트 스킬의 `allowed-tools` 는 workspace trust와 관계없이 적용된다. 신뢰하지 않은 폴더의 `-p` 실행에서도 마찬가지다. 즉 저장소에 커밋된 스킬이 스스로 넓은 도구 권한을 줄 수 있으므로, **다른 사람의 저장소에서 실행하기 전에 `.claude/skills` 의 `allowed-tools` 를 검토**한다. Windows에서는 셸 명령이 PowerShell 도구로 실행될 수 있으니 `Bash(...)` 와 `PowerShell(...)` 규칙을 함께 고려한다 ([2장](./02-workflow-and-settings.md)).

🔗 [Skills – pre-approve tools / restrict skill access](https://code.claude.com/docs/en/skills)

---

## 3-2. 스킬 사용과 개발

### 스킬 목록과 스킬 조합

**① 목록 확인**

- `/` 를 입력하면 명령과 스킬 목록이 나온다. `/skills` 로 스킬별 노출 상태를 관리한다.
- "What skills are available?" 처럼 물어보면 Claude가 아는 스킬 목록을 알려준다.

**② 호출 방법**

```text
/review-jsp src/main/webapp/WEB-INF/jsp/member       ← 메시지 맨 앞에서 직접 호출
좋아, 그럼 /deploy 로 스테이징에 배포해줘            ← 문장 중간: Claude가 요청 의도를 판단해 실행
/write-tests /fix-issue 123                         ← 체인: 맨 앞 스킬들을 함께 로드 (최대 6개)
```

**③ 조합 패턴 (공식 문서)**

| 패턴 | 예 |
|---|---|
| 스킬 + MCP | MCP가 DB에 연결하고, 스킬이 스키마와 쿼리 규칙을 알려준다 |
| 스킬 + 서브에이전트 | `/audit` 스킬이 보안·성능·스타일 서브에이전트를 띄운다 |
| `CLAUDE.md` + 스킬 | `CLAUDE.md` 는 "API 규칙을 따른다", 스킬은 전체 API 스타일 가이드 |

🔗 [Skills](https://code.claude.com/docs/en/skills) · [Combine features](https://code.claude.com/docs/en/features-overview#combine-features)

### 커스텀 스킬 개발

**① 개념** — 폴더를 만들고 `SKILL.md` 를 쓰면 바로 스킬이 된다. Claude에게 "이런 스킬 만들어줘"라고 요청해도 된다.

**② 왜 필요한가** — 반복 프롬프트를 표준화해 결과 품질을 일정하게 하고, 팀과 공유한다.

**③ 핵심 기법**

- **인자**: `$ARGUMENTS`, `$0`, 또는 `arguments: [file, level]` 로 이름 붙이기
- **동적 컨텍스트 주입**: 줄 맨 앞에 `` !`명령` `` 을 쓰면 스킬이 렌더링될 때 명령이 먼저 실행되고, Claude는 명령이 아닌 **실제 출력**을 본다. 실패(0이 아닌 종료 코드)하면 스킬 호출 자체가 중단된다. 권한 규칙 검사를 받으므로 `allowed-tools` 로 사전 승인해 둔다.

**④ 예시 – JSP 리뷰 스킬** (`.claude/skills/review-jsp/SKILL.md`, 규칙 내용은 예시)

```markdown
---
name: review-jsp
description: JSP와 Servlet/Controller 변경을 사내 보안·코딩 규칙으로 리뷰한다. JSP 파일이나 Controller 수정 후 리뷰를 요청받았을 때 사용한다.
argument-hint: [파일 또는 폴더]
allowed-tools: Bash(git diff *) Read Grep
---
# JSP 리뷰

## 변경된 파일
!`git diff --stat`

## 할 일
$ARGUMENTS 를 아래 기준으로 리뷰한다. 상세 기준은 checklist.md 를 읽는다.
1. 화면 출력값이 c:out 또는 fn:escapeXml 로 이스케이프되는가
2. 스크립틀릿 대신 JSTL/EL을 쓰는가
3. MyBatis에서 #{} 바인딩을 쓰는가 (${} 금지)
4. 로그인·권한 체크가 Controller에 있는가

## 출력 형식
- 지적마다 파일:라인, 문제, 수정안
- 코드에서 확인하지 못한 항목은 "확인 못 함"으로 표시한다
```

호출: `/review-jsp src/main/webapp/WEB-INF/jsp/member`

🔗 [Skills – arguments / inject dynamic context](https://code.claude.com/docs/en/skills)

### 스킬에서 환각 방지 지침 작성

> 🔎 **공식 문서 원칙에서 정리**: 공식 문서에 "환각 방지" 전용 절은 없다. 아래는 skills·best practices 문서의 원칙을 스킬 작성에 적용한 것이다.

| 원칙 | 스킬에서 적용하는 방법 | 근거 |
|---|---|---|
| 추측 대신 실제 데이터 | `` !`git diff` `` 처럼 동적 주입으로 실제 출력을 넣는다 | Claude는 명령이 아닌 실제 출력을 본다 |
| 참고 자료를 제공 | 규칙·스키마를 보조 파일로 두고 "`checklist.md` 를 읽어라"라고 지시 | 점진적 로딩 |
| 구체적이고 검증 가능한 지시 | "코드 품질 확인" 대신 "c:out 이스케이프 여부 확인" | 무엇을 할지 쓰라는 원칙 |
| 검증 단계 포함 | 마지막에 테스트·빌드를 실행하고 출력을 보여주게 한다 | best practices: 검증 수단 제공, 증거 요구 |
| 근거 표시 | 파일:라인을 쓰게 하고, 확인 못 한 것은 그렇다고 쓰게 한다 | (제안) 리뷰 결과 검증을 쉽게 함 |
| fork는 실행할 작업이 있을 때만 | 지침만 있는 스킬에 `context: fork` 를 쓰지 않는다 | 공식 경고: 작업이 없으면 의미 있는 결과 없이 끝남 |

🔗 [Skills](https://code.claude.com/docs/en/skills) · [Give Claude a way to verify its work](https://code.claude.com/docs/en/best-practices#give-claude-a-way-to-verify-its-work)

### 코드 실행 통합

**① 개념** — 스킬 폴더에 스크립트를 함께 넣고, 본문에서 실행하도록 지시한다. 반복적·결정적인 작업은 Claude가 매번 코드를 새로 쓰게 하지 말고 스크립트로 고정한다.

**② 방법** — 경로는 `${CLAUDE_SKILL_DIR}` 로 쓰고, `allowed-tools` 로 그 스크립트만 사전 승인한다.

```yaml
---
name: render-chart
description: Render a chart from CSV
allowed-tools: Bash(${CLAUDE_SKILL_DIR}/scripts/render.sh *)
---
Run `${CLAUDE_SKILL_DIR}/scripts/render.sh <csv-file>` to render the chart.
```

**③ Windows 참고** — 동적 주입 명령을 PowerShell로 실행하려면 frontmatter에 `shell: powershell` 을 쓴다 (PowerShell 도구가 켜져 있어야 함). `.ps1` 스크립트를 사전 승인할 때는 `PowerShell(...)` 규칙을 쓴다.

**④ 정책** — 관리자는 `disableSkillShellExecution: true` 로 스킬의 셸 주입을 막을 수 있다 (내장·managed 스킬은 제외).

🔗 [Skills – supporting files](https://code.claude.com/docs/en/skills) · [PowerShell tool](https://code.claude.com/docs/en/tools-reference#powershell-tool)

### 테스트와 검증 방법

| 확인할 것 | 방법 |
|---|---|
| 스킬이 인식되는가 | "What skills are available?" 로 목록에 있는지 확인 |
| 직접 호출이 되는가 | `/이름 인자` 로 실행 |
| 자동 호출(트리거)이 되는가 | description에 쓴 표현으로 자연스럽게 요청해 보기 |
| YAML 오류 | `claude plugin validate .claude/skills` (잘못된 YAML은 빈 메타데이터로 로드됨) |
| 스킬 효과를 수치로 | skill-creator 플러그인의 평가(스킬 있음/없음 비교), 플러그인에 넣은 스킬은 `claude plugin eval` 과 `tool_used: Skill` grader |

💡 플러그인 평가는 "모델이 바뀌거나 스킬을 고쳤을 때 성능이 떨어지지 않았는지"를 확인하고 CI에 연결하는 용도다.

🔗 [Skills – troubleshooting](https://code.claude.com/docs/en/skills) · [Test plugins with evals](https://code.claude.com/docs/en/plugin-evals)

---

## 3-3. 스킬 고급 사용과 최적화

### 동적 스킬 로딩 패턴

| 패턴 | 동작 | 언제 |
|---|---|---|
| 설명 기반 자동 로드 | 요청이 description과 맞으면 Claude가 본문을 로드 | 기본 |
| `paths` | 지정한 파일을 다룰 때만 자동 로드 (예: `"src/main/webapp/**/*.jsp"`) | JSP 규칙처럼 특정 파일 유형 전용 |
| 하위 폴더 발견 | 시작 폴더부터 저장소 루트까지의 `.claude/skills/` 를 모두 로드. 이름이 겹치면 하위 것은 `/하위경로:이름` | 멀티 모듈 저장소 |
| 실시간 반영 | 세션 중 `SKILL.md` 를 추가·수정하면 재시작 없이 반영. 새 최상위 skills 폴더는 `/reload-skills` | 스킬을 다듬는 중 |
| 동적 주입 | `` !`명령` `` 출력이 호출 시점마다 달라짐 | diff, 브랜치, 환경 정보 |

🔗 [Skills](https://code.claude.com/docs/en/skills)

### 스킬 성능 최적화

- **description**: 핵심 용도와 사용자가 실제로 쓸 단어를 앞에 둔다 (1,536자 제한). 너무 자주 호출되면 더 구체적으로 쓰거나 `disable-model-invocation: true`.
- **본문**: 500줄 이하, 할 일 중심, 중요한 지시는 앞쪽에 (압축 후 앞 5,000토큰만 유지).
- **보조 파일**: 상세 자료는 분리하고 본문에서 링크로 안내.
- **모델·노력**: 가벼운 스킬은 `model`, `effort` 로 조절할 수 있다.
- **격리**: 파일을 많이 읽는 조사형 스킬은 `context: fork` 로 메인 컨텍스트 밖에서 실행.

🔗 [Skills](https://code.claude.com/docs/en/skills)

### 스킬 버전 관리

> 🔎 **공식 문서 원칙에서 정리**: Claude Code 스킬 frontmatter에는 버전 필드가 없다 (`metadata` 는 자유 형식이라 직접 버전을 적어둘 수는 있지만 Claude Code는 읽지 않는다). 버전 관리는 배포 방식으로 한다.

- **프로젝트 스킬**: `.claude/skills/` 를 git으로 관리 → 커밋·리뷰·되돌리기가 그대로 버전 관리가 된다.
- **여러 저장소 공유**: 플러그인으로 묶어 마켓플레이스로 배포.
- **조직 표준**: managed 설정으로 배포.
- **이름이 겹칠 때 우선순위**: 엔터프라이즈 → 개인 → 프로젝트 / 내 스킬 → 내장 스킬 / 스킬 → `.claude/commands/` 파일.

🔗 [Skills – share skills](https://code.claude.com/docs/en/skills)

### 팀 스킬 개발

- 공유 방식 3가지 (공식 문서): **프로젝트 스킬 커밋**, **플러그인의 `skills/` 폴더**, **managed 배포**.
- 팀 규칙: 스킬도 코드처럼 PR로 리뷰한다. 특히 `allowed-tools` 와 `!` 주입 명령은 권한과 직결되므로 꼼꼼히 본다 (제안).
- 개인 실험 스킬은 `~/.claude/skills/` 에서 다듬은 뒤 프로젝트로 옮긴다 (제안).
- `claude.ai` 계정에 스킬을 올려 Cowork·클라우드 세션에서 쓰려면 Agent Skills 표준 필드(`name`, `description`, `allowed-tools`, `license`, `compatibility`, `metadata`)만 써야 업로드 오류가 나지 않는다.

🔗 [Skills – share skills](https://code.claude.com/docs/en/skills)

### 디버깅 기법

| 증상 | 확인·해결 |
|---|---|
| 스킬이 호출되지 않음 | 목록에 있는지 확인 → description에 사용자가 쓸 단어가 있는지 → YAML 유효성 (`claude plugin validate`) → 표현을 바꿔 요청 → `/이름` 으로 직접 호출 |
| 너무 자주 호출됨 | description을 구체적으로, 또는 `disable-model-invocation: true` |
| 도중에 지시를 안 따름 | 항상 지켜야 하는 규칙은 hook으로, 판단이 필요한 지침은 작업 전체에 적용되게 표현 ("편집할 때마다 테스트 실행") |
| 압축 후 지시가 흐려짐 | 스킬을 다시 호출해 전체 내용을 복원 |
| `!` 주입이 실패함 | 명령이 0이 아닌 코드로 끝나면 호출 전체가 중단됨 → 명령 단독 실행으로 확인, `allowed-tools` 승인 여부 확인 |
| 컨텍스트를 많이 씀 | `/context` 로 확인 → `skillOverrides`, 보조 파일 분리 |

🔗 [Skills – troubleshooting](https://code.claude.com/docs/en/skills)

### 재사용 가능한 스킬 패턴

| 패턴 | frontmatter 핵심 | 예 |
|---|---|---|
| 배경 지식 | `user-invocable: false` | 사내 Java 코딩 규칙 (Claude만 참조) |
| 경로 전용 규칙 | `paths` | JSP 파일 작업 시에만 로드되는 화면 규칙 |
| 수동 실행 작업 | `disable-model-invocation: true`, `allowed-tools` | `/commit`, `/deploy` |
| 격리 조사 | `context: fork`, `agent: Explore` | `/deep-dive 결제 모듈` |
| 스크립트 실행 | `scripts/`, `${CLAUDE_SKILL_DIR}` | 스크립틀릿 탐지 스크립트 |
| 외부 시스템 활용 | 스킬 + MCP | DB 스키마·쿼리 규칙 스킬 + DB MCP |

🔗 [Skills](https://code.claude.com/docs/en/skills) · [Combine features](https://code.claude.com/docs/en/features-overview#combine-features)

### 스킬 문서화 유의 사항

- **description = 무엇 + 언제**: Claude가 호출을 판단하는 유일한 근거다. 사용자가 실제로 쓸 단어를 넣는다.
- **본문은 간결하게**: 공식 문서는 `CLAUDE.md` 와 같은 "이 줄을 지우면 Claude가 실수하는가?" 기준을 적용하라고 한다.
- **보조 파일은 본문에서 안내**: "상세 API는 `reference.md` 참고"처럼 언제 읽을지 써준다.
- **경로는 변수로**: 스크립트 경로는 `${CLAUDE_SKILL_DIR}` 로 써야 어디에 설치돼도 동작한다.
- **부작용이 있으면 명시**: 배포·커밋처럼 상태를 바꾸는 스킬은 `disable-model-invocation: true` 로 사람이 실행하게 한다.

🔗 [Skills](https://code.claude.com/docs/en/skills)

### 스킬의 격리 실행: context: fork 메커니즘

**① 개념** — frontmatter에 `context: fork` 를 쓰면 Claude Code가 `agent` 에 지정한 종류의 **서브에이전트를 새로 띄우고 스킬 본문을 그 프롬프트로** 준다. 서브에이전트는 **현재 대화 기록을 보지 못하므로** 스킬 지시만으로 작업이 가능해야 한다.

**② 왜 쓰나** — 파일을 많이 읽는 조사·분석 결과만 메인 대화로 가져오고, 중간 과정은 메인 컨텍스트에 남기지 않는다.

**③ 동작**

| 항목 | 내용 |
|---|---|
| `agent` | `Explore`, `Plan`, `general-purpose`(기본), 또는 `.claude/agents/` 의 커스텀 에이전트 |
| 실행 방식 | 📌 **기본은 백그라운드**(v2.1.218부터). 결과가 끝나면 대화에 도착. `background: false` 면 그 턴에서 기다림 |
| 자동으로 기다리는 경우 | `-p`·Agent SDK, `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`, 같은 스킬이 아직 실행 중, 예약 작업 |
| 도구 | 백그라운드로 돌면 백그라운드 서브에이전트용 축소된 도구 세트. 필요한 도구가 빠지면 `background: false` |
| 되돌리기 | 백그라운드 fork의 파일 수정은 체크포인트 밖 → `/rewind` 로 되돌릴 수 없고 git으로 되돌린다 |

```yaml
---
name: deep-dive
description: 특정 모듈의 구조와 데이터 흐름을 깊이 조사한다
context: fork
agent: Explore
---
$ARGUMENTS 모듈을 조사한다:
1. Glob과 Grep으로 관련 파일을 찾는다
2. Controller → Service → DAO → 매퍼 흐름을 정리한다
3. 파일:라인 근거와 함께 요약한다
```

> ⚠️
> - 이름과 달리 `context: fork` 는 **"대화 fork"가 아니다**. 대화 기록이 필요하면 대화를 fork하는 기능([4장](./04-subagents.md))을 쓴다.
> - "API 규칙을 따른다" 같은 **지침만 있는 스킬에 쓰면** 서브에이전트가 할 일이 없어 의미 있는 결과 없이 끝난다.
> - 반대 방향은 서브에이전트의 `skills` 필드(스킬을 에이전트에 미리 주입)다 → [4장 "스킬 주입 예제"](./04-subagents.md#스킬-주입-예제).

🔗 [Skills – run skills in a subagent](https://code.claude.com/docs/en/skills)

---

## 핵심 정리

1. **스킬 = 필요할 때만 로드되는 지시·지식·절차 묶음.** 반복해서 붙여넣는 프롬프트가 생기면 스킬로 만든다.
2. **점진적 로딩 3단계**: 설명(항상) → 본문(호출 시) → 보조 파일(필요 시).
3. **설명은 매 요청 비용, 본문은 로드 후 계속 남는 비용.** description은 1,536자 안에서 핵심부터, `SKILL.md` 는 500줄 이하.
4. **참조형은 자동 호출, 작업형은 `disable-model-invocation: true`.**
5. **📌 `allowed-tools` 는 제한이 아니라 그 턴의 사전 승인이다.** 제한은 `disallowed-tools`, `Skill(...)` 권한 규칙, 서브에이전트 `tools`.
6. **남의 저장소에서는 커밋된 스킬의 `allowed-tools` 와 `!` 명령을 먼저 확인한다.**
7. **`!` 주입 명령으로 실제 데이터를 넣으면 추측이 줄어든다.**
8. **`context: fork` 는 격리 실행이며 기본이 백그라운드다.** 대화 기록을 보지 못하고, 수정은 `/rewind` 로 되돌릴 수 없다.
9. **팀 공유는 `.claude/skills` 커밋 → 플러그인 → managed 순으로 범위를 넓힌다.**

## 실무에서 바로 써먹기

**상황**: 팀에서 JSP 리뷰를 할 때마다 같은 체크리스트를 채팅에 붙여넣고 있다. 이를 스킬로 만들어 팀과 공유한다.

```text
[1. 스킬 초안 만들기]
  claude 실행 후: "JSP 리뷰 체크리스트를 .claude/skills/review-jsp 스킬로 만들어줘.
   상세 기준은 checklist.md 로 분리하고, 변경 파일 목록은 git diff --stat 으로 주입해줘."

[2. 내용 다듬기]
  description에 "JSP 리뷰", "Controller 변경 리뷰" 같은 실제 요청 표현을 넣는다
  SKILL.md 는 짧게, 사내 규칙 전문은 checklist.md 로

[3. 동작 확인]
  "What skills are available?" → 목록 확인
  /review-jsp src/main/webapp/WEB-INF/jsp/member → 직접 호출
  "방금 수정한 JSP 리뷰해줘" → 자동 호출되는지 확인

[4. 오류 점검]
  claude plugin validate .claude/skills
  /context 로 스킬 설명이 차지하는 양 확인

[5. 팀 공유]
  git add .claude/skills/review-jsp → PR로 리뷰 (allowed-tools 와 ! 명령 확인) → 머지

[6. 확장]
  파일을 많이 읽는 분석용은 context: fork + agent: Explore 버전을 따로 만든다
  리뷰 전담 서브에이전트에 이 스킬을 주입하는 방법은 4장 참고
```

## 참고한 공식 문서

- [Extend Claude with skills](https://code.claude.com/docs/en/skills)
- [Extend Claude Code (features overview)](https://code.claude.com/docs/en/features-overview)
- [Explore the context window](https://code.claude.com/docs/en/context-window)
- [Test plugins with evals](https://code.claude.com/docs/en/plugin-evals)
- [Configure permissions](https://code.claude.com/docs/en/permissions) · [Tools reference](https://code.claude.com/docs/en/tools-reference)
- [Agent Skills 표준](https://agentskills.io)
