# 📓 6장. 메모리와 대화 세션 관리

> **기준**: 「클로드코드로 시작하는 실전 에이전틱 코딩」 CHAPTER 06 목차 + Claude Code 공식 문서([code.claude.com/docs](https://code.claude.com/docs), 2026-10-11 확인). 책과 달라진 부분은 **📌 현재 공식 문서 기준**, 공식 문서에 전용 설명이 없는 부분은 **🔎 공식 문서 원칙에서 정리** 로 표시했다.
>
> 관련 노트: [1장](./01-getting-started.md) (컨텍스트 윈도우, `CLAUDE.md` 기초, 체크포인트 기초) · [2장](./02-workflow-and-settings.md) (세션 재개 명령, 설정 우선순위) · [5장](./05-output-interface.md) (출력 스타일과의 비교) · [7장](./07-automation.md) (훅)

---

## 6-1. 메모리 시스템

> `CLAUDE.md` 의 기본(`/init`, 200줄 원칙, 압축 후 재로드)은 [1장 컨텍스트 윈도우 관리 전략](./01-getting-started.md)에 정리했다. 여기서는 계층 구조, 로딩 규칙, 룰 시스템을 깊게 본다.

### 네 가지 계층적 메모리

**① 개념** — 세션은 항상 빈 컨텍스트로 시작한다. 세션을 넘어 지식을 전하는 장치는 두 가지다.

| | `CLAUDE.md` 파일 | auto memory |
|---|---|---|
| 누가 쓰나 | 사람 | Claude |
| 내용 | 지시와 규칙 | 배운 점과 패턴 |
| 범위 | 프로젝트, 사용자, 조직 | 저장소별 (워크트리끼리 공유) |
| 로드 | 매 세션 | 매 세션 (처음 200줄 또는 25KB) |

**② `CLAUDE.md` 4계층 (넓은 범위 → 좁은 범위, 로드 순서)**

| 계층 | 위치 | 용도 | 공유 |
|---|---|---|---|
| **관리형 정책** | Windows `C:\Program Files\ClaudeCode\CLAUDE.md` (macOS·Linux는 공식 문서 참고) | 회사 코딩 표준, 보안 정책, 컴플라이언스 | 조직 전체 |
| **사용자** | `~/.claude/CLAUDE.md` | 모든 프로젝트에 적용할 개인 선호 | 나만 |
| **프로젝트** | `./CLAUDE.md` 또는 `./.claude/CLAUDE.md` | 팀 공유 규칙, 아키텍처, 빌드 명령 | git으로 팀 공유 |
| **로컬** | `./CLAUDE.local.md` (`.gitignore` 에 추가) | 이 프로젝트의 내 개인 설정 | 나만 |

**③ 핵심 규칙**

- **덮어쓰기가 아니라 이어 붙이기**: 발견된 파일은 모두 컨텍스트에 연결된다. 나중에 로드된(작업 디렉터리에 가까운) 지시가 뒤에 온다.
- **충돌하면 아무거나 고를 수 있다**: 두 파일이 같은 동작에 다른 지시를 주면 Claude가 임의로 하나를 따를 수 있다. 계층끼리 일관되게 유지한다.
- **관리형 `CLAUDE.md` 는 개인 설정으로 제외할 수 없다.**
- `CLAUDE.md` 는 **시스템 프롬프트가 아니라 시스템 프롬프트 뒤의 사용자 메시지**로 전달된다. 엄격한 준수가 보장되지 않는다.

> 📌 **현재 공식 문서 기준 – `AGENTS.md` 지원 (v2.1.277+)**: 저장소에 `AGENTS.md` 만 있고 `CLAUDE.md`·`CLAUDE.local.md` 가 없으면 Claude가 `AGENTS.md` 를 프로젝트 지시로 읽는다. 둘 다 있으면 기본적으로 `CLAUDE.md` 만 읽는다. `/config` → **Project instructions** 로 "둘 다 읽기" 등을 고를 수 있다. 여러 AI 도구를 함께 쓰는 팀이면 `CLAUDE.md` 에 `@AGENTS.md` 를 import하는 방식이 가장 안전하다 (Windows에서는 심볼릭 링크 대신 import 권장).

🔗 [Choose where to put CLAUDE.md files](https://code.claude.com/docs/en/memory#choose-where-to-put-claude-md-files) · [AGENTS.md](https://code.claude.com/docs/en/memory#agents-md)

### 자동 메모리 (Auto memory)

**① 개념** — Claude가 작업하면서 **스스로** 남기는 메모. 기본으로 켜져 있다.

**② 무엇을 저장하나 (4종, frontmatter의 `type` 필드)**

| type | 내용 |
|---|---|
| `user` | 사용자의 역할, 전문성, 작업 선호 |
| `feedback` | 사용자가 준 교정, 확인해 준 방식 |
| `project` | 코드나 git 이력으로는 알 수 없는 진행 중인 작업, 마감, 결정 |
| `reference` | 프로젝트 밖 정보의 위치 (이슈 트래커, 대시보드 등) |

코드에서 알아낼 수 있는 것(구조, 파일 경로, 버그 수정 내용)과 `CLAUDE.md` 에 이미 있는 것은 저장하지 않는다. 매 세션 저장하지도 않는다.

**③ 저장 위치와 로딩**

```text
~/.claude/projects/<project>/memory/
├── MEMORY.md            # 색인. 한 줄에 메모 하나. 매 세션 로드 (처음 200줄 또는 25KB)
├── user_role.md         # 메모 하나 (시작 시 로드하지 않고 필요할 때 읽음)
└── feedback_testing.md
```

- `<project>` 는 **git 저장소 기준**이라 같은 저장소의 워크트리·하위 폴더가 한 메모리를 공유한다.
- **PC 로컬 전용**이다. 다른 PC나 클라우드 환경과 공유되지 않는다.
- 대화 기록 정리(`cleanupPeriodDays`) 대상에서 **제외**된다. 직접 지울 때까지 남는다.
- 메인 대화의 auto memory는 서브에이전트에 로드되지 않는다 (fork는 예외).

**④ 켜고 끄기**

| 방법 | 범위 |
|---|---|
| `/memory` → auto memory 토글 | 사용자 설정(`autoMemoryEnabled`)에 저장 |
| 프로젝트 설정에 `"autoMemoryEnabled": false` | 해당 프로젝트만 |
| 환경 변수 `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` | 전체 |
| `autoMemoryDirectory` 설정 | 저장 위치 변경 (절대 경로 또는 `~/` 로 시작) |

> ⚠️ **회사 소스 보안 관점**: auto memory는 내 PC의 `%USERPROFILE%\.claude\projects\…\memory\` 에 평문 Markdown으로 남는다. 고객 정보·접속 정보가 메모에 들어가지 않았는지 `/memory` 로 주기적으로 확인하고, 필요하면 프로젝트 단위로 끈다.

🔗 [Auto memory](https://code.claude.com/docs/en/memory#auto-memory)

### CLAUDE.md 가져오기와 조건부 로딩

**① `@` 가져오기 (import)**

```text
프로젝트 개요는 @README.md, 빌드 명령은 @pom.xml 참고.

# 추가 지시
- Git 규칙 @docs/git-instructions.md
- 개인 설정 @~/.claude/my-project-instructions.md
```

- 상대 경로는 **가져오는 파일 기준**으로 해석한다 (작업 디렉터리 기준 아님).
- 재귀 가져오기는 **최대 4단계**.
- 공백이 있는 경로는 공백 앞에 백슬래시: `@Design\ Docs/api.md`. 따옴표로 감싸면 가져오지 않는다.
- 코드 스팬·코드 블록 안의 `@경로` 는 가져오지 않는다 → 경로를 언급만 하려면 백틱으로 감싼다.
- **외부 가져오기**(작업 디렉터리 밖 경로)는 프로젝트에서 처음 만날 때 **승인 대화상자**가 뜬다. 남이 커밋한 파일로부터 나를 보호하기 위해서다.

**② 조건부 로딩 — 언제 로드되나**

| 위치 | 로드 시점 |
|---|---|
| 작업 디렉터리와 **상위** 디렉터리의 `CLAUDE.md`, `CLAUDE.local.md` | 시작할 때 |
| **하위** 디렉터리의 `CLAUDE.md` | Claude가 그 폴더의 파일을 읽거나 수정할 때 (온디맨드) |
| `paths` 없는 `.claude/rules/*.md` | 시작할 때 (`.claude/CLAUDE.md` 와 같은 우선순위) |
| `paths` 있는 룰 | 일치하는 파일을 Read·Write·Edit 할 때 |
| `--add-dir` 로 추가한 디렉터리의 `CLAUDE.md` | 기본은 로드 안 함. `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` 이면 로드 |

**③ 예시** — 멀티 모듈 Maven 프로젝트

```text
shop/
├── CLAUDE.md                  # 공통: 빌드 명령, 브랜치 규칙 (시작 시)
├── shop-web/CLAUDE.md         # JSP·Controller 규칙 (shop-web 파일을 만질 때)
└── shop-batch/CLAUDE.md       # 배치 규칙 (shop-batch 파일을 만질 때)
```

🔗 [Import additional files](https://code.claude.com/docs/en/memory#import-additional-files) · [How CLAUDE.md files load](https://code.claude.com/docs/en/memory#how-claude-md-files-load)

### 메모리 시스템을 자유자재로

**① 어디에 기록할지 고르기**

| 하고 싶은 일 | 방법 |
|---|---|
| 초안 만들기 | `/init` (기존 파일이 있으면 덮어쓰지 않고 개선안 제안) |
| 메모리 파일 열기·편집 | `/memory` (사용자·프로젝트 `CLAUDE.md`, `CLAUDE.local.md`, auto memory 폴더) |
| Claude가 기억하게 하기 | "항상 pnpm을 써. 기억해" → **auto memory** 에 저장 |
| `CLAUDE.md` 에 넣게 하기 | "이거 `CLAUDE.md` 에 추가해줘" 라고 명시 |
| 다른 도구 설정 옮겨오기 | `/init` 이 Cursor·Copilot 규칙을 읽어 반영, `/import` 로 다른 에이전트 설정 복사 (v2.1.213+) |
| 무엇이 로드됐는지 확인 | `/context` → **Memory files** |

**② 어떤 것을 `CLAUDE.md` 에 두나 (공식 기준)**

- **넣는다**: 매 세션 필요한 사실 — 빌드 명령, 규칙, 프로젝트 구조, "항상 X를 한다".
- **빼서 스킬로**: 여러 단계로 된 절차 ([3장](./03-agent-skills.md)).
- **빼서 경로별 룰로**: 코드베이스 일부에만 해당하는 규칙.
- **빼서 훅으로**: 특정 시점에 반드시 실행돼야 하는 것 ([7장](./07-automation.md)).

🔗 [View and edit with /memory](https://code.claude.com/docs/en/memory#view-and-edit-with-%2Fmemory) · [When to add to CLAUDE.md](https://code.claude.com/docs/en/memory#when-to-add-to-claude-md)

### 메모리 최적화와 장기 작업 관리

**① 크기 기준**

| 대상 | 기준 |
|---|---|
| `CLAUDE.md` 파일 하나 | **200줄 이하 권장**. 4MiB 넘으면 아예 건너뜀 |
| `MEMORY.md` | 처음 200줄 또는 25KB만 로드. 한도 가까이 가면 Claude Code가 Claude에게 줄이라고 알림 |
| 합계 | 각각은 짧아도 합계가 크면 시작 시 경고 (`CLAUDE.md`, 룰, import 각각 별도 파일로 계산) |

**② 줄이는 방법**

- 일부 영역에만 필요한 지시 → **`paths` 룰**로 옮긴다.
- 사람만 볼 메모 → 블록 HTML 주석(`<!-- … -->`)은 컨텍스트에 넣기 전에 제거되므로 토큰을 쓰지 않는다.
- 모노레포에서 다른 팀 `CLAUDE.md` 가 섞여 들어오면 → `claudeMdExcludes` 로 제외 (보통 `.claude/settings.local.json` 에).
- **`/doctor prompt-audit`** (v2.1.283+): 오래되거나 서로 충돌하는 지시, 존재하지 않는 파일 참조를 찾아 수정안을 제안한다.

**③ 장기 작업에서 지시가 사라질 때**

- 압축(`/compact`) 후 프로젝트 루트 `CLAUDE.md` 는 디스크에서 다시 읽는다.
- 하위 폴더 `CLAUDE.md`, `paths` 룰은 다시 해당 파일을 만질 때 로드된다.
- **대화로만 준 지시는 사라질 수 있다** → 계속 지켜야 하면 `CLAUDE.md` 로 옮긴다.

**④ 지시를 안 따를 때 점검 순서 (공식 문서)**

1. `/context` 에서 해당 파일이 **Memory files** 에 있는지
2. 로드되는 위치에 있는지
3. 구체적인지 ("코드 정리" 대신 "들여쓰기 4칸")
4. 다른 파일과 충돌하지 않는지
5. Claude Code 자체 지시와 겹치는지 (커밋·PR 규칙이면 `includeGitInstructions`, `attribution` 설정 확인)
6. 그래도 안 되면 → **훅**으로 강제하거나, 시스템 프롬프트 수준이 필요하면 `--append-system-prompt`

🔗 [Write effective instructions](https://code.claude.com/docs/en/memory#write-effective-instructions) · [Troubleshoot memory issues](https://code.claude.com/docs/en/memory#troubleshoot-memory-issues)

### 모듈형 룰 시스템 (`.claude/rules/`)

**① 개념** — 지시를 **주제별 파일**로 나눈다. 하위 폴더까지 재귀로 찾는다.

```text
.claude/
├── CLAUDE.md
└── rules/
    ├── code-style.md
    ├── testing.md
    ├── security.md
    └── backend/
        └── mybatis.md
```

**② 경로별 룰 (`paths` frontmatter)** — 일치하는 파일을 다룰 때만 로드된다.

```markdown
---
paths:
  - "src/main/webapp/**/*.jsp"
  - "src/main/webapp/**/*.{js,css}"
---
# JSP 규칙
- 스크립틀릿(<% %>) 대신 JSTL/EL을 쓴다
- 출력은 <c:out> 또는 fn:escapeXml 로 이스케이프한다
- 인코딩은 UTF-8, 페이지 지시자를 바꾸지 않는다
```

- `paths` 는 룰이 읽는 **유일한 필드**다. 다른 필드는 에러 없이 무시된다.
- YAML 파싱이 실패하면 `paths` 가 없는 것처럼 **항상 로드**된다 → `claude --debug` 로 확인.
- `{a,b}` 중괄호 확장은 룰 하나당 1,000개 패턴 / 4MiB 예산 안에서만 동작한다.
- 파일 이름의 `[` 를 그대로 맞추려면 `\[` 로 이스케이프한다.

**③ 룰 vs 스킬** — 룰은 매 세션 또는 일치하는 파일을 열 때 로드된다. 작업할 때만 필요한 지시는 **스킬**이 낫다.

**④ 프로젝트끼리 룰 공유** — `.claude/rules/` 는 심볼릭 링크를 지원한다. 단, 링크 대상이 작업 디렉터리 밖이면 외부 import처럼 **승인 후에만** 로드되고, 승인 뒤에도 `paths` 없는 룰만 로드된다. 네트워크 경로(UNC `\\server\share`)를 가리키는 링크는 로드하지 않는다.

🔗 [Organize rules with .claude/rules/](https://code.claude.com/docs/en/memory#organize-rules-with-claude%2Frules%2F) · [Path-specific rules](https://code.claude.com/docs/en/memory#path-specific-rules)

### 사용자 수준 룰과 고급 기능

**① 사용자 수준 룰** — `~/.claude/rules/` 의 룰은 내 PC의 모든 프로젝트에 적용된다.

- 프로젝트 룰보다 **먼저** 로드된다. 서로 덮어쓰지 않으므로 충돌하면 Claude가 아무거나 따를 수 있다.
- 공유 룰을 승인 없이 쓰고 싶으면 심볼릭 링크 대신 여기에 둔다.

**② 조직 수준**

- 관리형 `CLAUDE.md` 파일 배포 (MDM, 그룹 정책 등) 또는 관리형 설정의 `claudeMd` 키에 내용을 직접 넣는다.
- 역할 구분 (공식 표):

| 목적 | 둘 곳 |
|---|---|
| 도구·명령·경로 차단 | 관리형 설정 `permissions.deny` |
| 샌드박스 강제 | 관리형 설정 `sandbox.enabled` |
| 코드 스타일, 데이터 취급 주의, 행동 지침 | 관리형 `CLAUDE.md` |

설정은 **클라이언트가 강제**하고, `CLAUDE.md` 는 **행동을 유도**할 뿐이다.

**③ 디버깅용 고급 기능**

- **`InstructionsLoaded` 훅**: 어떤 `CLAUDE.md`·룰이 언제, 왜 로드됐는지 기록한다 (경로별 룰이 안 먹을 때 유용).
- **서브에이전트 전용 메모리**: 서브에이전트 frontmatter의 `memory` 필드로 별도 auto memory를 가질 수 있다 ([4장](./04-subagents.md)).
- **`CLAUDE_CODE_NEW_INIT=1`**: `/init` 을 여러 단계로 묻는 대화형 흐름으로 바꾼다.

🔗 [User-level rules](https://code.claude.com/docs/en/memory#user-level-rules) · [Manage CLAUDE.md for large teams](https://code.claude.com/docs/en/memory#manage-claude-md-for-large-teams) · [InstructionsLoaded](https://code.claude.com/docs/en/hooks#instructionsloaded)

---

## 6-2. 체크포인팅과 복구

> 체크포인트 개념, 기본 메뉴, git `reset`/`revert` 와의 차이는 [1장 체크포인트와 되감기](./01-getting-started.md)에 정리했다. 여기서는 메뉴의 세부 동작, 한계, 실험 워크플로를 다룬다.

### /rewind 화면

**① 여는 방법** — `/rewind`, 또는 입력창이 **비어 있을 때** `Esc` 두 번. (입력이 있으면 `Esc` 두 번은 입력을 지우고 기록에 저장 → `Up` 으로 복구)

**② 화면 구성** — 세션에서 보낸 프롬프트 목록이 나온다. 지점을 고른 뒤 동작을 고른다.

| 동작 | 결과 | 끝난 뒤 |
|---|---|---|
| Restore code and conversation | 코드·대화 모두 그 시점으로 | 원래 프롬프트가 입력창에 복원됨 |
| Restore conversation | 대화만 되돌림, 코드는 현재 상태 유지 | 원래 프롬프트가 입력창에 복원됨 |
| Restore code | 코드만 되돌림, 대화 유지 | |
| Summarize from here | 이 지점 **이후**를 요약 | 원래 프롬프트가 입력창에 복원됨 |
| Summarize up to here | 이 지점 **이전**을 요약, 이후 메시지는 그대로 | 대화 끝, 입력창 비어 있음 |
| Never mind | 아무것도 안 함 | |

**③ 세부 동작**

- 코드 복원 메뉴 2개는 그 체크포인트 이후 **추적된 파일 변경이 있을 때만** 나온다.
- 요약 옵션은 화살표로 선택한 뒤 **add context (optional)** 칸에 요약 방향을 적을 수 있다. 숫자 키로 바로 고르면 지시 없이 요약한다.
- 요약해도 파일은 그대로이고, 원본 메시지는 세션 기록에 남아 Claude가 참조할 수 있다. 대화에는 **Summarized conversation** 표시가 남는다.
- `/clear` 전 대화는 같은 프로세스 안이면 목록 맨 위 `/resume <session-id> (previous session)` 항목으로 돌아갈 수 있다.

🔗 [Rewind and summarize](https://code.claude.com/docs/en/checkpointing#rewind-and-summarize)

### 제한 사항과 대응

| 한계 | 대응 |
|---|---|
| **Bash로 바뀐 파일**(`rm`, `mv`, `cp`, 빌드 산출물)은 추적 안 됨 | 파괴적 명령은 권한으로 묻게 두고, 작업 전 커밋 |
| **서브에이전트 편집**은 대부분 복원 안 됨 | git으로 되돌린다. 예외: **포그라운드에서 실행된 `context: fork` 스킬**은 내 턴 안에서 편집하므로 복원된다 |
| 백그라운드 fork 스킬(기본값), 백그라운드 `/code-review --fix` | 복원 안 됨 → git |
| 외부 편집, 다른 세션의 편집 | 보통 추적 안 됨 → git |
| **작업 중에 보낸 메시지**는 자기 체크포인트가 없음 (**No code restore** 표시) | 그 턴을 시작한 프롬프트를 골라 Restore code (턴 전체 편집이 되돌아감) |
| **심볼릭·하드 링크 파일**은 건너뜀 (`Restored the code, but skipped N files` 경고) | 복원 전에 `/debug` 를 켜면 `~/.claude/debug/<session-id>.txt` 에 건너뛴 경로가 남음 |
| 스냅샷은 약 30일 뒤 정리 (`No files were restored`) | 오래 보관하려면 `cleanupPeriodDays` |
| DB·API·배포 같은 원격 작업 | 되돌릴 수 없음 → 권한으로 통제 |

> 💡 체크포인트는 **세션당 최근 100개**의 파일 스냅샷을 보관한다. 오래된 체크포인트가 버려지면 남은 체크포인트가 참조하지 않는 스냅샷은 삭제된다.

🔗 [Limitations](https://code.claude.com/docs/en/checkpointing#limitations)

### 체크포인팅과 깃

공식 문서 입장은 하나다: **체크포인트는 세션 수준의 빠른 복구용이고, 영구 이력과 협업은 git으로 한다.**

> 🔎 **공식 문서 원칙에서 정리**: 아래 역할 분담은 공식 원칙("not a replacement for version control")을 실무 흐름으로 풀어 쓴 것이다.

| 시점 | 체크포인트 | git |
|---|---|---|
| 작업 시작 전 | (자동) | 작업 트리를 깨끗하게: 커밋 또는 stash, 브랜치 생성 |
| 한 턴 안의 시행착오 | **Restore code** 로 즉시 되돌리기 | — |
| 방향이 맞다고 확인됨 | — | **작은 단위로 커밋** (체크포인트가 못 잡는 Bash 변경까지 보호) |
| Claude가 실행한 `git commit` 을 취소 | 불가 (셸 명령) | `git reset` / `git revert` |
| 세션 밖으로 결과 공유 | 불가 (내 PC 로컬) | push, PR |

**(제안)** "테스트가 통과할 때마다 커밋" 규칙을 `CLAUDE.md` 에 두면 체크포인트(편집 단위)와 커밋(검증된 단위)이 자연스럽게 나뉜다.

🔗 [Not a replacement for version control](https://code.claude.com/docs/en/checkpointing#not-a-replacement-for-version-control)

### 다중 체크포인트와 스마트 복구

> 🔎 **공식 문서 원칙에서 정리**: "스마트 복구"는 공식 용어가 아니다. 프롬프트마다 체크포인트가 쌓이고, 코드와 대화를 따로 되돌릴 수 있다는 공식 동작을 조합한 복구 패턴이다.

| 상황 | 고를 것 |
|---|---|
| 마지막 변경만 틀렸다 | 직전 프롬프트 → **Restore code**, 대화는 남겨 "왜 틀렸는지" 이어서 설명 |
| 몇 턴 전부터 방향이 틀렸다 | 틀어지기 시작한 프롬프트 → **Restore code and conversation** → 복원된 프롬프트를 고쳐 다시 보냄 |
| 코드는 좋은데 대화가 실패한 시도로 가득하다 | **Restore conversation** (코드 유지) 또는 실패 구간을 **Summarize from here** |
| 초반 지시는 지키고 중간 디버깅 로그만 줄이고 싶다 | 디버깅 시작 지점 → **Summarize from here** + "원인과 해결만 남겨" |
| 오래된 앞부분만 압축하고 최근 흐름은 유지 | **Summarize up to here** |

**주의**: Restore code는 **추적된 편집만** 되돌린다. 그 사이 Bash로 만든 파일, `mvn` 빌드 산출물, Claude가 실행한 git 명령은 그대로 남는다 → 복원 후 `git status` 로 확인한다.

🔗 [Common use cases](https://code.claude.com/docs/en/checkpointing#common-use-cases)

### 실험 주도 개발 워크플로

> 🔎 **공식 문서 원칙에서 정리**: "실험 주도 개발"은 공식 용어가 아니다. 공식 문서의 체크포인트 사용 사례("다른 구현 방식 탐색"), `/branch`, 워크트리를 조합했다.

**① 세 가지 실험 도구**

| 도구 | 무엇을 분리하나 | 언제 |
|---|---|---|
| 체크포인트 (`/rewind`) | 같은 세션·같은 파일에서 **순차적으로** 시도 | 짧은 시도를 빠르게 바꿔 볼 때 |
| `/branch` 또는 `claude --continue --fork-session` | **대화**를 복사해 갈라짐 (원본 세션은 그대로) | 같은 맥락에서 다른 접근을 따로 이어 갈 때 |
| `claude --worktree 이름` | **파일**을 별도 작업 디렉터리·브랜치로 분리 | 두 구현을 동시에 실제로 돌려 볼 때 |

> ⚠️ `/branch` 는 대화만 갈라진다. 파일은 같은 작업 트리를 쓰므로, 두 분기에서 같은 파일을 고치면 서로 덮어쓴다. 파일까지 분리하려면 워크트리를 쓴다.

**② 흐름 예시**

```text
1. git switch -c exp/paging        # 깨끗한 상태에서 실험 브랜치
2. "MyBatis RowBounds로 페이징 구현해줘" → 테스트
3. 마음에 안 듦 → /rewind → 2번 프롬프트 → Restore code and conversation
4. "이번엔 LIMIT/OFFSET 쿼리로 구현해줘" → 테스트 → 비교
5. 두 방식을 동시에 보고 싶다 → claude --worktree paging-b (다른 터미널)
6. 결정 → 이긴 쪽 커밋, 진 쪽 워크트리는 종료 시 제거
```

🔗 [Checkpointing](https://code.claude.com/docs/en/checkpointing) · [Branch a session](https://code.claude.com/docs/en/sessions#branch-a-session) · [Worktrees](https://code.claude.com/docs/en/worktrees)

---

## 6-3. 세션 관리 전략

> `--continue`, `--resume`, `-n`, `/rename`, `/branch` 명령 표와 `-p` 에서 이어가기는 [2장 비대화형 모드와 세션 재개](./02-workflow-and-settings.md)에 정리했다. 여기서는 재개 동작의 세부와 운영 전략을 다룬다.

### 컨텍스트 윈도우 최적화와 세션 복구

**① 재개하면 무엇이 돌아오나 (공식 문서)**

| 복원됨 | 복원 안 됨 |
|---|---|
| 도구 호출·결과를 포함한 전체 대화 기록 | `--mcp-config`, `--settings`, `--plugin-dir`, `--add-dir` 같은 **실행 플래그** → 다시 넘긴다 |
| 모델, `--agent` 로 시작한 에이전트 | 세션 중 `/add-dir` 로 추가한 디렉터리 |
| 권한 모드 (터미널에서 `--continue` / 이름·ID로 재개할 때) | 백그라운드 Bash·모니터 작업 |
| 만료되지 않은 예약 작업 | 중단된 시점에 실행 중이던 도구 (다시 실행하지 않음) |

> 📌 **현재 공식 문서 기준 – 권한 모드 복원의 예외**: `bypassPermissions` 로 끝난 세션은 재개해도 **바이패스가 복원되지 않는다** (실행할 때 다시 켜야 함). plan 모드로 끝난 세션은 plan 모드로 돌아온다.

**② 요약으로 재개하기 (Pro·Max)**

약 **1시간 넘게 쉬었고 10만 토큰이 넘는** 세션을 재개하면, 첫 메시지 전에 대화상자가 뜬다. 프롬프트 캐시가 만료되어 전체 기록을 다시 보내는 비용이 크기 때문이다.

| 선택 | 결과 |
|---|---|
| Resume from summary | 즉시 `/compact` → 이후 요청이 저렴해짐 (요약에서 빠진 세부는 잃음) |
| Resume full session as-is | 그대로 로드 (요청마다 대화 크기만큼 비용) |
| Don't ask me again | 항상 전체 재개 |

**③ 세션 안에서 컨텍스트 다루기** — `/clear`(새로 시작, 이전 세션은 저장됨), `/compact [지시]`, `/context` ([1장](./01-getting-started.md) 참고).

🔗 [What a resumed session restores](https://code.claude.com/docs/en/sessions#what-a-resumed-session-restores) · [Resume from a summary](https://code.claude.com/docs/en/sessions#resume-from-a-summary)

### 다중 세션 관리

**① 이름 붙이기** — 이름을 붙여야 찾고 재개하기 쉽다.

| 시점 | 방법 |
|---|---|
| 시작할 때 | `claude -n order-refund` |
| 세션 중 | `/rename order-refund` |
| 세션 선택 화면에서 | `Ctrl+R` |
| plan 승인 시 | 이름이 없으면 계획 기반 제목이 자동 생성 |

이름이 없으면 첫 프롬프트를 요약한 **자동 제목**이 생기고, 그 제목으로도 `--resume` 할 수 있다.

**② 세션 선택 화면 (`claude --resume`, `/resume`)**

| 키 | 동작 |
|---|---|
| `Space` | 세션 내용 미리보기 |
| `Ctrl+B` | 현재 git 브랜치 세션만 |
| `Ctrl+W` | 저장소의 모든 워크트리 세션 |
| `Ctrl+A` | 이 PC의 모든 프로젝트 세션 |
| `/` 또는 글자 입력 | 검색 (PR URL을 붙여 넣으면 그 PR을 만든 세션을 찾음) |

**③ 병렬 작업**

- **서로 다른 작업** → 터미널마다 다른 이름의 세션.
- **같은 파일을 건드리는 병렬 작업** → `claude --worktree 이름` 으로 파일까지 분리.
- **같은 세션을 두 터미널에서 fork 없이 재개하면** 두 쪽 메시지가 한 기록에 섞인다 → 갈라지려면 `/branch` 또는 `--fork-session`.

> 📌 **현재 공식 문서 기준**: `-p`, Agent SDK, `/loop` 로 시작한 세션은 세션 선택 화면과 `claude --continue` 에 나오지 않는다. 세션 ID로 `--resume` 한다. `--from-pr <번호>` 는 해당 PR과 연결된 세션만 보여 준다.

🔗 [Name your sessions](https://code.claude.com/docs/en/sessions#name-your-sessions) · [Use the session picker](https://code.claude.com/docs/en/sessions#use-the-session-picker) · [Worktrees](https://code.claude.com/docs/en/worktrees)

### 세션 데이터 보관

**① 저장 위치** — `~/.claude/projects/<project>/<session-id>.jsonl` (Windows: `%USERPROFILE%\.claude\projects\…`). `<project>` 는 작업 디렉터리 경로의 영숫자 아닌 문자를 `-` 로 바꾼 이름이다.

> ⚠️ JSONL 내부 형식은 **버전마다 바뀔 수 있다.** 스크립트로 직접 파싱하지 말고 아래 공식 인터페이스를 쓴다.

**② 사람이 읽을 기록 vs 스크립트용 데이터**

| 목적 | 방법 |
|---|---|
| 사람이 읽을 기록 | `/export` (클립보드 또는 텍스트 파일), `/export 파일명` |
| 실행 결과를 구조화된 데이터로 | `claude -p … --output-format json` |
| 기존 세션에 질문해서 답 받기 | `claude -p --resume <id> --output-format json "변경 내용 요약해줘"` |
| 세션 종료 시 자동 보관 | `SessionEnd` 훅에서 입력의 `transcript_path` 를 복사 ([7장](./07-automation.md)) |

**③ 보관 기간과 저장 제어**

| 하고 싶은 일 | 설정 |
|---|---|
| 보관 기간 변경 (기본 30일) | `cleanupPeriodDays` |
| 저장 위치 이동 | `CLAUDE_CONFIG_DIR` |
| 모든 모드에서 기록 저장 안 함 | `CLAUDE_CODE_SKIP_PROMPT_HISTORY` |
| `-p` 실행 한 번만 저장 안 함 | `--no-session-persistence` |
| 프로젝트 기록을 지금 지우기 | `claude purge` |

> 💡 **회사 소스 보안 관점**: 대화 기록에는 Claude가 읽은 소스와 명령 출력이 그대로 들어간다. 민감한 프로젝트는 `cleanupPeriodDays` 를 짧게 두고, 프로젝트를 마치면 `claude purge` 로 정리하는 것을 검토한다. 단, 기간을 줄이면 체크포인트 스냅샷도 함께 빨리 정리된다.

🔗 [Export and locate session data](https://code.claude.com/docs/en/sessions#export-and-locate-session-data) · [Where transcripts are stored](https://code.claude.com/docs/en/sessions#where-transcripts-are-stored)

### 협업에서의 세션 관리

> 🔎 **공식 문서 원칙에서 정리**: 공식 문서에 "팀 세션 공유" 기능은 없다. 세션은 **각자 PC에 저장**되고, 팀과 공유되는 것은 git에 올린 파일뿐이라는 사실에서 정리했다.

| 공유하고 싶은 것 | 방법 |
|---|---|
| 프로젝트 규칙, 반복 설명 | `CLAUDE.md`, `.claude/rules/` 커밋 |
| 반복 작업 절차 | 스킬 커밋 ([3장](./03-agent-skills.md)) |
| 이번 작업의 결정과 맥락 | `/export` 결과나 Claude가 쓴 요약을 PR 설명·이슈에 첨부 |
| PR 리뷰 후속 작업 | `claude --from-pr <번호>` 로 내가 그 PR을 만든 세션을 찾아 이어감 |
| 작업 체크리스트 | `CLAUDE_CODE_TASK_LIST_ID=이름` → `~/.claude/tasks/` 의 같은 목록을 여러 세션이 공유 (내 PC 안) |

**(제안)** 세션 기록 원문을 그대로 공유하지 않는다. 읽은 소스·로그가 모두 들어 있으므로, 요약본만 공유한다.

🔗 [Manage sessions](https://code.claude.com/docs/en/sessions) · [Task list](https://code.claude.com/docs/en/interactive-mode#task-list)

### 장기 프로젝트 세션 전략

> 🔎 **공식 문서 원칙에서 정리**: 공식 best practices의 "관련 없는 작업은 `/clear`", "계속 필요한 지시는 `CLAUDE.md`" 원칙과 세션 기능을 조합했다.

1. **세션 = 작업 하나.** 이름을 이슈 번호·기능명으로 (`claude -n JIRA-1234`).
2. **작업 전환 = `/clear` 또는 새 세션.** 관련 없는 작업을 한 세션에 섞지 않는다.
3. **며칠 걸리는 작업** → 같은 이름으로 `claude -r JIRA-1234`. 오래 쉬었다면 "Resume from summary" 로 비용을 줄인다.
4. **세션이 사라져도 남아야 하는 것** → `CLAUDE.md`(규칙), auto memory(선호·교정), git 커밋(코드), PR 설명(결정).
5. **실행 플래그에 의존하는 세션** (`--add-dir`, `--mcp-config`) → 재개 명령을 메모해 두거나 설정 파일로 옮긴다.
6. **보관 기간보다 오래 갈 작업** → 중요한 결정은 세션이 아니라 문서·커밋 메시지에 남긴다 (기본 30일 후 기록 삭제).

🔗 [Best practices: Manage your session](https://code.claude.com/docs/en/best-practices#manage-your-session)

---

## 핵심 정리

1. **`CLAUDE.md` 는 4계층**(관리형 → 사용자 → 프로젝트 → 로컬)이고, **덮어쓰지 않고 이어 붙는다.** 충돌하면 아무거나 따를 수 있다.
2. **auto memory는 Claude가 쓰는 메모**다. `MEMORY.md` 앞 200줄/25KB만 로드되고, PC 로컬에만 있으며, 기록 정리 대상에서 제외된다.
3. **상위 폴더 `CLAUDE.md` 는 시작 시, 하위 폴더는 그 폴더 파일을 만질 때** 로드된다. `paths` 룰도 일치하는 파일을 만질 때 로드된다.
4. **`@경로` import는 최대 4단계**, 외부 경로는 처음에 승인을 받는다.
5. **200줄 이하, 구체적으로, 일부 영역용은 룰로, 절차는 스킬로, 강제는 훅으로.**
6. **되감기 메뉴는 코드와 대화를 따로** 되돌리고, 요약은 "여기부터/여기까지"로 범위를 고른다.
7. **체크포인트가 못 잡는 것**: Bash 변경, 대부분의 서브에이전트 편집, 외부 편집, 링크 파일, 원격 작업 → git과 권한으로 보완한다.
8. **실험은 3단계로**: 순차 시도는 체크포인트, 대화 분기는 `/branch`, 파일 분리는 `--worktree`.
9. **재개 시 실행 플래그와 bypass 모드는 복원되지 않는다.** 오래 쉰 큰 세션은 요약으로 재개해 비용을 줄인다.
10. **세션은 내 PC에만 있다.** 팀과 공유할 것은 `CLAUDE.md`·룰·스킬·커밋·PR로 옮긴다.

## 실무에서 바로 써먹기

**상황**: Windows + IntelliJ, Spring + MyBatis + JSP 멀티 모듈 Maven 프로젝트. 3일에 걸쳐 "주문 환불 기능"을 개발한다. 팀원 4명이 같은 저장소를 쓴다. (경로·이름은 예시)

```text
[1. 메모리 계층 정리 (팀)]
  ./CLAUDE.md           : 빌드(mvn -q -pl shop-web test), 브랜치 규칙, 금지 사항 (200줄 이하)
  .claude/rules/jsp.md  : paths "shop-web/src/main/webapp/**/*.jsp" → JSP 이스케이프 규칙
  .claude/rules/mybatis.md : paths "**/mapper/**/*.xml" → #{} 사용, ${} 금지
  → 커밋. 개인 DB 접속 URL은 CLAUDE.local.md (.gitignore)

[2. 1일차 시작]
  git switch -c feature/refund
  claude -n JIRA-1234
  /context 로 Memory files 에 CLAUDE.md, 룰이 로드됐는지 확인

[3. 실험]
  "환불 금액 계산을 서비스 레이어에 구현해줘" → 테스트 실패, 방향이 틀림
  Esc Esc → 해당 프롬프트 → Restore code and conversation → 프롬프트 수정 후 재전송
  테스트 통과 → 커밋 (체크포인트가 못 잡는 mvn 산출물·Bash 변경까지 보호)

[4. 긴 디버깅 후 컨텍스트 정리]
  /rewind → 디버깅 시작 지점 → Summarize from here + "원인과 수정 내용만 남겨"

[5. 2–3일차 이어가기]
  claude -r JIRA-1234 → "Resume from summary" 선택 (하루 쉬었고 대화가 큼)

[6. 반복 교정은 메모리로]
  같은 지적을 두 번 했다 → "이 규칙 CLAUDE.md 에 추가해줘" (팀 규칙이면)
  개인 선호면 "기억해" → auto memory

[7. 마무리와 공유]
  /export refund-session.txt → 요약만 PR 설명에 정리 (원문 기록은 공유하지 않음)
  PR 리뷰 지적 수정은 claude --from-pr 123 으로 해당 세션 재개
```

## 참고한 공식 문서

- [How Claude remembers your project](https://code.claude.com/docs/en/memory) · [Explore the context window](https://code.claude.com/docs/en/context-window)
- [Checkpointing](https://code.claude.com/docs/en/checkpointing)
- [Manage sessions](https://code.claude.com/docs/en/sessions) · [Worktrees](https://code.claude.com/docs/en/worktrees) · [Best practices](https://code.claude.com/docs/en/best-practices)
- [Hooks reference: InstructionsLoaded](https://code.claude.com/docs/en/hooks#instructionsloaded) · [Interactive mode: Task list](https://code.claude.com/docs/en/interactive-mode#task-list)
