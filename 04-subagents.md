# 📕 4장. 서브에이전트

> **기준**: 「클로드코드로 시작하는 실전 에이전틱 코딩」 CHAPTER 04 목차 + Claude Code 공식 문서([code.claude.com/docs](https://code.claude.com/docs), 2026-10-04 확인). 책과 달라진 부분은 **📌 현재 공식 문서 기준**, 공식 문서에 전용 설명이 없는 부분은 **🔎 공식 문서 원칙에서 정리** 로 표시했다.
>
> 관련 노트: [1장](./01-getting-started.md) (컨텍스트) · [2장](./02-workflow-and-settings.md) (권한) · [3장](./03-agent-skills.md) (스킬)

---

## 4-1. 서브에이전트의 개념과 아키텍처

**서브에이전트란** — 특정 작업을 맡는 전문 AI 도우미. 각자 **별도 컨텍스트 윈도우**, 전용 시스템 프롬프트, 제한된 도구, 독립된 권한으로 실행되고 결과 **요약만** 돌려준다. 사용량은 메인 대화와 같은 한도에서 차감된다.

**공식 문서가 말하는 이점**: 컨텍스트 보존, 도구 제한으로 제약 강제, 설정 재사용, 전문화된 프롬프트, 저렴한 모델(Haiku 등)로 비용 조절.

### 에이전트 위임 패턴: 오케스트레이터-워커

> 🔎 **공식 문서 원칙에서 정리**: "오케스트레이터-워커"는 공식 문서 용어가 아니다. 공식 문서의 "메인 대화가 위임하고 서브에이전트가 요약을 돌려준다"는 구조에 대응시킨 것이다.

**① 개념** — 메인 대화(오케스트레이터)가 작업을 나눠 서브에이전트(워커)에게 맡기고, 결과를 받아 종합한다.

**② 왜** — 워커가 수십 개 파일을 읽어도 메인에는 요약만 들어온다.

**③ Claude Code에서** — Claude가 `Agent` 도구로 서브에이전트를 띄운다. description을 보고 **자동으로 위임**하거나, 사용자가 명시적으로 지정한다.

**④ 규모별 선택 (공식 비교)**

| | 서브에이전트 | 에이전트 팀 | 동적 워크플로 |
|---|---|---|---|
| 누가 다음 작업을 정하나 | Claude가 턴마다 | 리더 에이전트가 턴마다 | Claude가 쓴 스크립트 |
| 중간 결과 | Claude의 컨텍스트 | 공유 태스크 목록 | 스크립트 변수 |
| 규모 | 턴당 몇 개 | 오래 도는 동료 몇 명 | 실행당 수십–수백 개 |

🔗 [Subagents](https://code.claude.com/docs/en/sub-agents) · [When to use a workflow](https://code.claude.com/docs/en/workflows#when-to-use-a-workflow)

### 단일 책임 원칙과 서브에이전트

> 🔎 **공식 문서 원칙에서 정리**: 공식 문서는 "특정 작업 유형을 맡는 전문 도우미", "집중된 시스템 프롬프트로 행동을 전문화"라고 설명한다.

- **한 에이전트 = 한 역할**: 리뷰어, 테스트 실행기, DB 조회기처럼 나눈다.
- **역할에 맞는 도구만**: 리뷰어는 `Read, Grep, Glob` 만 주면 실수로 코드를 고칠 수 없다.
- **description은 구체적으로**: "Code reviewer"(모호) 대신 "코드 수정 후 품질·보안을 리뷰한다. 코드 변경 뒤 use proactively".

🔗 [Subagents](https://code.claude.com/docs/en/sub-agents)

### 컨텍스트 엔지니어링 원칙

**① 서브에이전트가 시작할 때 받는 것 / 받지 않는 것 (공식 문서)**

| 받는 것 | 받지 않는 것 |
|---|---|
| 자기 시스템 프롬프트 + 환경 정보 (Claude Code 기본 프롬프트가 아님) | **메인 대화 기록** |
| Claude가 쓴 위임 프롬프트 | 메인에서 이미 호출한 스킬 |
| `CLAUDE.md` 계층, git status (Explore·Plan은 둘 다 생략) | 메인이 이미 읽은 파일 |
| `skills` 필드로 지정한 스킬 전체 내용 | auto memory, output style |

**② 원칙**

- 위임 프롬프트에 **필요한 맥락(파일 위치, 목표, 완료 기준)을 담는다** — 대화 기록이 넘어가지 않기 때문.
- 대화 맥락이 꼭 필요하면 **fork**(대화를 통째로 복사한 서브에이전트)를 쓴다. fork는 메인과 프롬프트 캐시를 공유해 새 서브에이전트보다 저렴하다.
- 이미 대화에 있는 내용에 대한 질문이면 서브에이전트 대신 `/btw` (도구 없음, 기록에 안 남음).
- 서브에이전트도 메인과 같은 규칙으로 자동 압축되고, 트랜스크립트는 별도 파일로 저장된다.

🔗 [Subagents – what loads at startup / fork the current conversation](https://code.claude.com/docs/en/sub-agents)

### 에이전트 체인과 병렬 실행

```text
# 순차 체인
code-reviewer 서브에이전트로 성능 문제를 찾고, optimizer 서브에이전트로 고쳐줘

# 병렬 조사
인증, 주문, 결제 모듈을 서브에이전트 3개로 병렬 조사해줘

# 대량 출력 격리
서브에이전트로 전체 테스트를 돌리고 실패한 테스트와 에러 메시지만 보고해줘
```

> 📌 **현재 공식 문서 기준**
> - 서브에이전트가 **자기 서브에이전트를 띄울 수 있다** (기본 3단계 깊이, `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`).
> - 동시 실행은 기본 **20개**까지 (`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`).
> - 대화형 세션은 fork 모드가 켜져 있어 Claude가 띄운 서브에이전트가 **기본적으로 백그라운드**에서 돈다.

🔗 [Subagents – common patterns](https://code.claude.com/docs/en/sub-agents)

### 언제 서브에이전트를 사용하는가

| 선택 | 이럴 때 (공식 문서) |
|---|---|
| 메인 대화 | 주고받기가 잦을 때, 여러 단계가 맥락을 많이 공유할 때, 작은 수정, 지연이 중요할 때 |
| 서브에이전트 | 다시 안 볼 대량 출력이 생길 때, 도구·권한을 제한하고 싶을 때, 독립 작업으로 요약만 받으면 될 때, 병렬 실행 |
| 스킬 | 메인 대화 안에서 쓰는 재사용 지시·절차 ([3장](./03-agent-skills.md)) |
| `/btw` | 이미 대화에 있는 내용에 대한 짧은 질문 |

🔗 [Subagents](https://code.claude.com/docs/en/sub-agents)

---

## 4-2. 서브에이전트 사용하기

### 사용 가능한 서브에이전트 목록

**내장 서브에이전트**

| 이름 | 특징 |
|---|---|
| `Explore` | 빠른 **읽기 전용** 코드 탐색 (Write·Edit 불가). `CLAUDE.md`·git status 생략. 재개 불가(1회성) |
| `Plan` | Plan 모드에서 계획 전 조사. 읽기 전용, 재개 불가 |
| `general-purpose` | 탐색과 수정이 모두 필요한 복잡한 다단계 작업. 모든 도구 |
| 기타 | `claude`(범용), `statusline-setup`, `claude-code-guide`(Claude Code 사용법 질문) |

**내 서브에이전트**: `.claude/agents/`(프로젝트), `~/.claude/agents/`(개인), 플러그인, managed.

> 📌 **현재 공식 문서 기준**: `/agents` 는 더 이상 대화형 마법사가 아니다(v2.1.198부터). 실행하면 "Claude에게 요청하거나 `.claude/agents/` 를 직접 편집하라"는 안내만 나온다. 실행 중인 서브에이전트는 `/tasks` 로 본다.

🔗 [Subagents – built-in subagents](https://code.claude.com/docs/en/sub-agents)

### 에이전트 호출 문법

| 방법 | 예 | 보장 |
|---|---|---|
| 자동 위임 | "이 변경 리뷰해줘" → description이 맞는 에이전트에 위임 | Claude 판단 |
| 자연어로 지정 | "java-reviewer 서브에이전트로 최근 변경을 봐줘" | 대체로 |
| @-멘션 | `@agent-java-reviewer 인증 변경 봐줘` (플러그인은 `@agent-플러그인:이름`) | **그 에이전트 실행 보장** |
| 세션 전체 | `claude --agent java-reviewer` (4-5 참고) | 메인 스레드가 그 에이전트가 됨 |

🔗 [Subagents – invoke explicitly](https://code.claude.com/docs/en/sub-agents)

### 에이전트 결과 통합

| | 포그라운드 | 백그라운드 |
|---|---|---|
| 메인 대화 | 끝날 때까지 기다림 | 계속 작업 가능, 끝나면 결과 도착 |
| 도구 | 전체 | 축소된 내장 도구 세트 + 모든 MCP 도구 |
| 조작 | `Ctrl+B` 로 백그라운드 전환 | `/tasks` 에서 트랜스크립트 보기, `x` 로 중지 |

- 결과는 요약으로 메인에 들어오고, Claude가 여러 결과를 종합한다.
- 보안: 서브에이전트 보고서에 지시처럼 보이는 문구가 섞이면 Claude Code가 이를 표시한다. 보고서를 근거로 한 도구 호출도 일반 권한 검사를 그대로 거친다.

🔗 [Subagents – foreground or background](https://code.claude.com/docs/en/sub-agents)

### 작업 재개 기능 활용

**① 개념** — 서브에이전트는 끝나면 **에이전트 ID**를 돌려준다. Claude가 `SendMessage` 도구로 그 ID(또는 이름)에 메시지를 보내면 이전 대화·도구 결과를 그대로 가진 채 이어서 작업한다.

**② 예시**

```text
java-reviewer 서브에이전트로 인증 모듈을 리뷰해줘
[완료]
그 리뷰 이어서 이번엔 권한 체크 로직을 분석해줘     ← 같은 에이전트가 맥락을 유지한 채 재개
```

**③ 주의**

- `Explore`, `Plan` 은 1회성이라 재개할 수 없다 → 이어서 해야 하면 `general-purpose` 나 커스텀 에이전트.
- 사용자가 직접 중지한(`/tasks` 에서 `x`) 에이전트는 자동 재개되지 않는다.
- 트랜스크립트 위치: `~/.claude/projects/프로젝트/세션ID/subagents/agent-에이전트ID.jsonl` (기본 30일 보관).

📌 `SendMessage` 는 에이전트 팀을 켜지 않아도 쓸 수 있다.

🔗 [Subagents – resume subagents](https://code.claude.com/docs/en/sub-agents)

### 에러 처리 및 복구

| 상황 | 동작·대응 (공식 문서) |
|---|---|
| `maxTurns` 도달 | 결과가 **부분(partial)** 으로 표시되고, Claude가 메시지를 보내 이어서 진행할 수 있다 |
| `tools` 가 하나도 해석되지 않음 | 서브에이전트가 실행되지 않고 오류 ("zero tools") → 도구 이름 확인 |
| 이름에 `:` 포함 | 파일이 로드되지 않음 (`:` 는 플러그인 범위용) |
| `--agent` 로 재개했는데 에이전트가 사라짐 | 기본 도구로 계속하고 경고 표시 |
| 백그라운드 작업이 실패 | `/tasks` 에서 트랜스크립트 확인 → 재개하거나 다시 위임 |
| 서브에이전트가 고친 파일을 되돌리고 싶음 | 대부분의 서브에이전트 편집은 체크포인트로 되돌릴 수 없다 → git |

🔗 [Subagents](https://code.claude.com/docs/en/sub-agents) · [Checkpointing](https://code.claude.com/docs/en/checkpointing)

### 서브에이전트 성능 측정

> 🔎 **공식 문서 원칙에서 정리**: 서브에이전트 전용 성능 지표 기능은 공식 문서에 없다. 아래는 공식 기능으로 확인할 수 있는 범위다.

- **진행·결과**: `/tasks` 에서 실행 중·완료된 서브에이전트와 트랜스크립트 확인.
- **비용**: `/usage` (세션 비용·플랜 한도). 서브에이전트 요청도 같은 한도에서 차감된다. `-p` 에서는 `--output-format json` 의 `total_cost_usd`.
- **컨텍스트**: 메인 `/context` 가 줄어드는지로 위임 효과를 확인.
- **비용 조절 수단**: 에이전트별 `model: haiku`, `effort`, `maxTurns`, 또는 `CLAUDE_CODE_SUBAGENT_MODEL` 로 일괄 지정.

🔗 [Subagents – choose a model](https://code.claude.com/docs/en/sub-agents) · [Manage costs](https://code.claude.com/docs/en/costs)

---

## 4-3. 커스텀 서브에이전트 개발과 스킬 주입

### 서브에이전트 정의 파일 구조

**① 구조** — 마크다운 파일 하나 = 에이전트 하나. YAML frontmatter(설정) + 본문(시스템 프롬프트). 하위 폴더로 정리할 수 있다 (`agents/review/security.md`).

**② 범위와 우선순위** (이름이 같으면 위가 이김)

| 우선순위 | 위치 | 범위 |
|---|---|---|
| 1 | managed 설정 | 조직 전체 |
| 2 | `--agents` CLI JSON | 이번 세션 |
| 3 | `.claude/agents/` | 프로젝트 (커밋해서 팀 공유) |
| 4 | `~/.claude/agents/` | 내 모든 프로젝트 |
| 5 | 플러그인 `agents/` | 플러그인이 켜진 곳 |

**③ 만드는 법** — Claude에게 요청하거나 파일을 직접 작성한다.

```text
~/.claude/agents/ 에 code-improver 개인 서브에이전트를 만들어줘.
읽기 전용으로, 가독성·성능 개선점을 찾고 현재 코드와 개선안을 보여주게 해줘. Sonnet을 써줘.
```

🔗 [Subagents – choose the subagent scope](https://code.claude.com/docs/en/sub-agents)

### YAML 프론트매터 완전 가이드

| 필드 | 필수 | 설명 |
|---|---|---|
| `name` | ✅ | 고유 이름. `:` 사용 불가, 파일명과 달라도 됨 |
| `description` | ✅ | 언제 위임할지. 자동 위임의 판단 기준 |
| `tools` | | 쓸 수 있는 도구 목록 (생략 시 전부 상속) |
| `disallowedTools` | | 뺄 도구. `Bash(git push *)` 처럼 써도 도구 전체가 빠짐 |
| `model` | | `sonnet`, `opus`, `haiku`, `fable`, 전체 모델 ID, `inherit` |
| `permissionMode` | | `default`, `acceptEdits`, `auto`, `dontAsk`, `bypassPermissions`, `plan` (메인이 bypass·acceptEdits·auto면 무시됨) |
| `maxTurns` | | 최대 턴 수, 도달하면 부분 결과 |
| `skills` | | 시작 시 **전체 내용을 주입**할 스킬 |
| `mcpServers` | | 이 에이전트만 쓸 MCP 서버 (이름 참조 또는 인라인 정의) |
| `hooks` | | 이 에이전트가 활성일 때만 도는 hook |
| `memory` | | 세션 간 기억: `user`, `project`(팀 공유 권장), `local` |
| `background` | | `true` 면 항상 백그라운드 |
| `isolation` | | `worktree` 면 임시 git worktree에서 실행 (변경 없으면 자동 정리) |
| `effort` | | 노력 수준 |
| `omitClaudeMd` | | `true` 면 사용자·프로젝트·로컬 `CLAUDE.md` 생략 |
| `initialPrompt` | | `--agent` 로 메인 세션이 될 때 자동으로 보내는 첫 메시지 |
| `color` | | 표시 색상 |

> 📌 **현재 공식 문서 기준**: `memory`, `isolation`, `background`, `effort`, `omitClaudeMd`, `initialPrompt`, 인라인 `mcpServers` 는 비교적 최근에 추가된 필드다. 플러그인 서브에이전트에서는 `permissionMode`, `mcpServers`, `hooks`, `initialPrompt` 가 무시된다. 서브에이전트 도구 이름은 `Agent` 이고, 권한 규칙도 `Agent(이름)` 으로 쓴다.

🔗 [Subagents – supported frontmatter fields](https://code.claude.com/docs/en/sub-agents)

### 프롬프트 작성 가이드

- **description**: 무엇을 하는지 + 언제 쓰는지. 자동 위임을 원하면 "use proactively"를 넣는다. 전체 서브에이전트 설명 합계가 **15,000토큰을 넘으면 시작 시 경고**가 뜬다.
- **본문(시스템 프롬프트)**: 역할, 작업 절차, 출력 형식을 쓴다. 메인 대화 기록이 없으므로 필요한 규칙은 본문이나 `skills` 로 준다.
- **memory 활용**: "작업 전에 메모리를 확인하고, 끝나면 배운 패턴을 메모리에 저장하라"를 본문에 넣으면 스스로 기억을 쌓는다.

```text
✓ Reviews code for quality and best practices. Use after writing or modifying code.
✗ Code reviewer      (너무 모호)
```

🔗 [Subagents – automatic delegation / persistent memory](https://code.claude.com/docs/en/sub-agents)

### 에이전트 테스트와 최적화

- **호출 확인**: `@agent-이름` 으로 확실히 실행해 결과를 본 뒤, 자연어 요청으로 자동 위임이 되는지 확인.
- **도구 최소화**: 읽기만 하면 `Read, Grep, Glob`. 위험한 명령은 에이전트 전용 `PreToolUse` hook으로 검사 (공식 예: 읽기 전용 DB 조회기).
- **모델 선택**: 단순 조사는 `haiku` 로 비용 절감. 우선순위: 호출 시 지정 → 정의의 `model` → `CLAUDE_CODE_SUBAGENT_MODEL` → 메인 모델.
- **격리**: 파일을 고치는 에이전트를 병렬로 돌릴 때는 `isolation: worktree`.
- **턴 제한**: 폭주를 막으려면 `maxTurns`.
- **개선 루프**: 결과가 기대와 다르면 description과 본문을 고쳐 다시 실행 (파일 수정은 다음 실행부터 반영).

🔗 [Subagents – example subagents](https://code.claude.com/docs/en/sub-agents)

### 스킬 주입 예제

**① 개념** — `skills` 필드에 적은 스킬은 서브에이전트 시작 시 **전체 내용이 주입**된다 (설명만이 아님). 실행 중에 스킬을 찾아 로드할 필요가 없다.

**② 3장 `context: fork` 와의 관계**

| | 에이전트의 `skills` 필드 | 스킬의 `context: fork` |
|---|---|---|
| 주체 | 에이전트가 스킬을 지식으로 가져감 | 스킬이 에이전트를 실행 환경으로 씀 |
| 프롬프트 | 에이전트 본문 + 위임 프롬프트 | 스킬 본문이 곧 프롬프트 |
| 어울리는 스킬 | 참조형 (규칙·지식) | 작업형 (명시적 절차) |

**③ 예시** — 참조형 스킬 `java-conventions` 를 리뷰 에이전트에 주입 (내용은 예시)

```markdown
---
name: java-reviewer
description: Java/JSP 변경을 보안·성능·사내 규칙 관점에서 리뷰한다. 코드 수정 후 커밋 전에 use proactively.
tools: Read, Grep, Glob, Bash
model: sonnet
skills:
  - java-conventions
memory: project
---
너는 Java/JSP 시니어 리뷰어다. 코드를 수정하지 않는다.
1. git diff 로 변경 파일을 확인한다
2. 주입된 java-conventions 규칙과 보안(XSS, SQL 바인딩, 권한 체크) 기준으로 검토한다
3. 지적마다 파일:라인, 심각도, 수정안을 쓴다
4. 작업 전 메모리에서 이 프로젝트의 반복 이슈를 확인하고, 끝나면 새로 발견한 패턴을 저장한다
```

**④ 주의**

- `disable-model-invocation: true` 인 스킬은 주입할 수 없다 (내장 `/verify` 포함).
- `skills` 는 "미리 넣을 스킬"일 뿐 접근 제한이 아니다. 스킬 사용 자체를 막으려면 `tools` 에서 `Skill` 을 빼거나 `disallowedTools` 에 넣는다.
- 지정한 스킬이 없거나 비활성이면 건너뛰고 디버그 로그에 경고만 남는다.

🔗 [Subagents – preload skills into subagents](https://code.claude.com/docs/en/sub-agents)

---

## 4-4. 에이전트 팀

> 📌 **현재 공식 문서 기준 – 실험 기능, 기본 꺼짐.** `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 을 환경 변수나 settings의 `env` 에 설정해야 한다. 세션 재개·태스크 조정·종료 동작에 알려진 제약이 있다. `-p` (비대화형)에서는 팀원이 생성되지 않는다.

### 에이전트 팀의 핵심 개념

**① 개념** — 여러 Claude Code 세션이 팀으로 일한다. **리더**(메인 세션)가 작업을 나누고 종합하며, **팀원**은 각자 컨텍스트에서 일하고 **서로 직접 메시지**를 주고받는다. 사용자도 팀원에게 직접 말할 수 있다.

**② 구성 요소**

| 요소 | 역할 |
|---|---|
| 리더 | 팀원 생성·조정·종합 (세션 동안 고정) |
| 팀원 | 독립된 Claude Code 인스턴스 |
| 태스크 목록 | 공유 작업 목록 (가져가기·완료) |
| 메일박스 | 에이전트 간 메시지 (`~/.claude/teams/팀이름/inboxes/`) |

**③ 잘 맞는 작업 (공식 문서)**: 여러 관점의 조사·리뷰, 독립된 새 모듈, 가설 경쟁 디버깅, 프론트·백엔드·테스트처럼 계층을 나눈 작업. 순차 작업·같은 파일 수정·의존성 많은 작업에는 맞지 않는다.

🔗 [Agent teams](https://code.claude.com/docs/en/agent-teams)

### 팀 생성과 생애 주기 관리

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

```text
# 생성 (자연어)
PR #142를 리뷰할 팀원 3명을 만들어줘: 보안 담당, 성능 담당, 테스트 커버리지 담당.
각자 리뷰하고 결과를 보고하게 해줘.

# 종료
security 팀원에게 종료하라고 해줘
```

- **생성**: Claude가 이름을 붙여 서브에이전트를 띄우면 팀원으로 실행된다. 확인 없이 생성되며, 팀 기능이 켜져 있으면 요청하지 않아도 팀이 생길 수 있다.
- **표시 모드**: 기본 `in-process`(아무 터미널, 프롬프트 아래 패널에서 `↑↓` 선택·`Enter` 로 보기·`x` 중지·`Ctrl+T` 태스크 목록). 분할 화면은 tmux·iTerm2 필요 — **Windows Terminal과 VS Code 통합 터미널은 분할 화면 미지원**.
- **모델**: 생성 프롬프트에서 지정 → 서브에이전트 정의의 `model` → `CLAUDE_CODE_SUBAGENT_MODEL` → 리더 모델.
- **계획 먼저**: 리더를 Plan 모드로 두고 팀원을 만들면 팀원도 계획 승인 전까지 읽기 전용.
- **종료**: 이름으로 종료 요청 → 팀원이 승인하거나 이유와 함께 거부. 세션이 끝나면 팀 디렉터리는 자동 정리.
- **크기**: 공식 권장은 **3–5명**, 팀원당 태스크 5–6개.

🔗 [Agent teams – start your first team / control your team](https://code.claude.com/docs/en/agent-teams)

### 컨텍스트와 권한

- **컨텍스트**: 팀원은 일반 세션처럼 `CLAUDE.md`, MCP, 스킬을 로드하고 리더의 생성 프롬프트를 받는다. **리더의 대화 기록은 넘어가지 않는다** → 생성 프롬프트에 작업 세부 정보를 담는다.
- **권한**: 팀원은 리더의 권한 모드로 시작한다 (`dontAsk` 는 상속 안 됨). 팀원의 권한 확인 창은 **리더 세션에 뜬다** → 자주 쓰는 명령은 미리 allow 규칙에 넣어 둔다. 생성 시점에 팀원별 모드를 따로 줄 수는 없다.
- **보안**: 다른 에이전트가 보낸 메시지는 **사용자의 승인으로 인정되지 않는다**. 거부된 작업을 다른 팀원에게 넘겨 우회할 수도 없다.
- **비용**: 팀원마다 별도 컨텍스트라 토큰이 팀원 수에 비례해 늘어난다.

🔗 [Agent teams – permissions / context and communication](https://code.claude.com/docs/en/agent-teams)

### 메시지 전달 시스템

- 팀원끼리 **이름으로** 직접 메시지를 보낸다 (`SendMessage`). 전원에게 보내려면 받는 사람마다 한 번씩 보낸다.
- 메시지는 자동 전달되어 리더가 확인하러 다닐 필요가 없다.
- 팀원이 일을 마치면 리더에게 **유휴 알림**과 최종 답을 자동으로 보낸다.
- 받는 사람의 메일박스 파일에 쓰기가 성공해야 "보냄"으로 처리된다.
- 생성할 때 팀원 이름을 정해 주면 이후 프롬프트에서 부르기 쉽다.

🔗 [Agent teams – context and communication](https://code.claude.com/docs/en/agent-teams)

### 태스크 조정 시스템

- 공유 태스크 상태: **pending → in progress → completed**. 의존 태스크가 끝나야 다음 태스크를 가져갈 수 있다(자동 해제).
- 배정: 리더가 지정하거나, 팀원이 끝나면 다음 미배정·비차단 태스크를 **스스로 가져간다**. 동시에 가져가는 충돌은 파일 잠금으로 막는다.
- 품질 관문 hook: `TeammateIdle`(유휴 직전), `TaskCreated`, `TaskCompleted` — exit code 2로 막고 피드백을 줄 수 있다.
- 알려진 문제: 태스크 상태가 늦게 갱신될 수 있다 → 실제로 끝났는지 확인하고 리더에게 재촉.

🔗 [Agent teams – assign and claim tasks / quality gates with hooks](https://code.claude.com/docs/en/agent-teams)

### 파일 소유권 전략

- 공식 문서: **두 팀원이 같은 파일을 수정하면 덮어쓰기가 생긴다 → 팀원마다 다른 파일 묶음을 맡긴다.**
- Java/JSP 예 (제안): 팀원 A는 `src/main/java/.../member/**`, B는 `src/main/webapp/WEB-INF/jsp/member/**`, C는 `src/test/**` 처럼 경로로 나눠 생성 프롬프트에 명시한다.
- 처음에는 코드 작성 없이 **리뷰·조사**부터 시작하라고 공식 문서가 권한다.

🔗 [Agent teams – best practices](https://code.claude.com/docs/en/agent-teams)

### 서브에이전트와 에이전트 팀

| | 서브에이전트 | 에이전트 팀 |
|---|---|---|
| 컨텍스트 | 자기 창, 결과는 호출자에게 | 자기 창, 완전히 독립 |
| 소통 | 호출자에게 결과 반환 (이름 있는 서브에이전트끼리 메시지 가능) | 팀원끼리 직접 메시지 |
| 조정 | 메인이 모든 작업 관리 | 메시지 + 공유 태스크 목록으로 자율 조정 |
| 적합 | 결과만 중요한 집중 작업 | 토론·협업이 필요한 복잡한 작업 |
| 토큰 비용 | 낮음 | 높음 |

**팀의 제약 (공식 문서)**: `/resume`·`/rewind` 가 in-process 팀원을 복원하지 않음, 세션당 팀 하나, 팀원은 팀원을 만들 수 없음(중첩 불가), 리더 교체 불가, 종료가 느릴 수 있음.

💡 팀이 켜진 상태에서 원치 않게 팀원이 생기면 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` 를 `0` 으로 바꾼다 (세션 재시작 불필요).

🔗 [Agent teams – compare with subagents / limitations](https://code.claude.com/docs/en/agent-teams)

---

## 4-5. CLI 기반 동적 에이전트 활용

### --agent 플래그: 기존 에이전트 직접 실행

**① 개념** — `claude --agent 이름` 으로 시작하면 **메인 스레드 자체가 그 에이전트**가 된다 (도구 제한·모델 적용).

```powershell
claude --agent java-reviewer
```

**② 특징 (공식 문서)**

- 커스텀 에이전트의 본문이 Claude Code 기본 시스템 프롬프트를 **통째로 대체**한다 (`--system-prompt` 와 같은 방식). 단 `CLAUDE.md` 와 프로젝트 메모리는 그대로 로드된다.
- 시작 화면에 `@이름` 이 표시된다. 세션을 재개해도 에이전트 설정이 유지된다.
- 프로젝트 기본값으로 쓰려면 `.claude/settings.json` 에 `"agent": "java-reviewer"` (CLI 플래그가 우선).
- `initialPrompt` 를 정의해 두면 시작하자마자 첫 작업을 자동으로 보낸다.

**③ 즉석 정의 – `--agents`** (이번 세션 한정, 우선순위 2위)

```bash
claude --agents '{"code-reviewer":{"description":"Expert code reviewer. Use proactively after code changes.","prompt":"You are a senior code reviewer.","tools":["Read","Grep","Glob","Bash"],"model":"sonnet"}}'
```

💡 PowerShell에서는 JSON 따옴표 처리가 번거롭다. `-p` 실행이면 JSON 파일 경로를 넘길 수 있다: `claude -p --agents .\agents.json "변경 리뷰해줘"` (v2.1.281 이상).

🔗 [Subagents – run the whole session as a subagent](https://code.claude.com/docs/en/sub-agents) · [CLI flags](https://code.claude.com/docs/en/cli-reference#cli-flags)

### 에이전트 해석 과정

이름으로 에이전트를 찾을 때의 규칙 (공식 문서):

1. 같은 이름이면 우선순위: **managed → `--agents` → `.claude/agents/` → `~/.claude/agents/` → 플러그인**
2. 프로젝트 에이전트는 현재 폴더에서 저장소 루트까지 올라가며 찾고, 같은 이름이 여러 개면 **현재 폴더에 가장 가까운 것**을 쓴다.
3. `--add-dir` 로 추가한 폴더의 `.claude/agents/` 도 로드된다.
4. 플러그인 에이전트는 이름만으로도 찾지만, 겹치면 `플러그인:이름` (하위 폴더면 `플러그인:폴더:이름`).
5. 프로젝트 에이전트 파일의 인라인 MCP 서버는 그 폴더를 **trust한 뒤에만** 로드된다.

🔗 [Subagents – choose the subagent scope](https://code.claude.com/docs/en/sub-agents)

### CLI 에이전트와 에이전트 팀의 관계

**① 정의 재사용** — 서브에이전트 정의(security-reviewer 등)를 팀원 역할로 그대로 쓸 수 있다: "security-reviewer 에이전트 타입으로 팀원을 만들어 인증 모듈을 감사해줘".

**② 팀원에게 적용되는 것 / 안 되는 것**

| 정의 항목 | 팀원에게 |
|---|---|
| `tools`, `model` | 적용 (in-process는 `SendMessage`·태스크 도구가 자동 추가) |
| 본문 | in-process: 기본 프롬프트 뒤에 추가 / 분할 화면: 기본 프롬프트를 대체 |
| `skills` | **적용 안 됨** (팀원은 프로젝트·사용자 스킬을 로드) |
| `mcpServers` | 분할 화면만 적용, in-process는 무시 |

**③ 정리** — 한 번 정의한 역할을 ① 위임용 서브에이전트, ② `--agent` 로 실행하는 메인 세션, ③ 팀원 세 가지 방식으로 재사용할 수 있다.

🔗 [Agent teams – use subagent definitions for teammates](https://code.claude.com/docs/en/agent-teams)

---

## 핵심 정리

1. **서브에이전트 = 별도 컨텍스트에서 일하고 요약만 돌려주는 전문 도우미.** 다시 안 볼 대량 출력·조사를 맡긴다.
2. **서브에이전트는 메인 대화 기록을 받지 않는다.** 위임 프롬프트에 맥락을 담고, 대화 맥락이 꼭 필요하면 fork를 쓴다.
3. **정의 파일 = YAML frontmatter + 시스템 프롬프트.** `name`·`description` 필수, `tools` 로 역할에 맞게 제한한다.
4. **description이 자동 위임을 결정한다.** "언제 쓰는지"와 "use proactively"를 넣는다.
5. **📌 `/agents` 마법사는 없어졌다.** Claude에게 만들어 달라고 하거나 `.claude/agents/` 를 직접 편집한다.
6. **재개는 에이전트 ID + `SendMessage`.** Explore·Plan은 재개할 수 없다.
7. **`skills` 필드는 스킬을 에이전트에 주입하고, `context: fork` 는 스킬을 에이전트에서 실행한다.**
8. **대화형 세션의 서브에이전트는 기본 백그라운드이며 중첩(3단계)이 가능하다.**
9. **에이전트 팀은 실험 기능이고 비용이 크다.** 3–5명, 파일 소유권을 나누고, 리뷰·조사부터 시작한다.
10. **`--agent` 는 메인 세션을 그 에이전트로 바꾼다.** 같은 정의를 서브에이전트·메인 세션·팀원으로 재사용할 수 있다.

## 실무에서 바로 써먹기

**상황**: 회사 Java/JSP 프로젝트에서 커밋 전 리뷰를 전담할 서브에이전트를 만들고, 큰 조사는 병렬로 맡긴다. (이름·경로는 예시)

```text
[1. 규칙 스킬 준비 (3장)]
  .claude/skills/java-conventions/SKILL.md  ← 참조형, 사내 Java/JSP 규칙

[2. 리뷰 서브에이전트 만들기]
  "java-reviewer 프로젝트 서브에이전트를 만들어줘. 읽기 전용(Read, Grep, Glob, Bash),
   sonnet, java-conventions 스킬 주입, memory: project, 커밋 전 use proactively."
  → .claude/agents/java-reviewer.md 검토 후 커밋

[3. 큰 조사는 병렬로]
  "회원, 주문, 결제 모듈의 트랜잭션 처리 방식을 서브에이전트 3개로 병렬 조사하고
   공통 문제만 요약해줘"

[4. 구현 후 리뷰 위임]
  @agent-java-reviewer 방금 변경한 내용 리뷰해줘

[5. 이어서 리뷰]
  "그 리뷰 이어서 예외 처리 부분도 봐줘"   ← 같은 에이전트 재개 (SendMessage)

[6. 진행 확인]
  /tasks 로 백그라운드 서브에이전트 확인, /usage 로 비용 확인

[7. (선택) 리뷰 전용 세션]
  claude --agent java-reviewer

[8. (실험) 여러 관점 동시 리뷰가 필요할 때만]
  CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 → 보안·성능·테스트 팀원 3명으로 PR 리뷰
```

## 참고한 공식 문서

- [Create custom subagents](https://code.claude.com/docs/en/sub-agents)
- [Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams)
- [Orchestrate subagents at scale with dynamic workflows](https://code.claude.com/docs/en/workflows)
- [Extend Claude Code (features overview)](https://code.claude.com/docs/en/features-overview)
- [CLI reference](https://code.claude.com/docs/en/cli-reference) · [Configure permissions](https://code.claude.com/docs/en/permissions)
- [Checkpointing](https://code.claude.com/docs/en/checkpointing) · [Manage costs](https://code.claude.com/docs/en/costs)
