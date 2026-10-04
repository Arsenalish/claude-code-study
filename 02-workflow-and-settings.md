# 📘 2장. Claude Code 워크플로와 설정

> **기준**: 「클로드코드로 시작하는 실전 에이전틱 코딩」 02장 목차 + Claude Code 공식 문서([code.claude.com/docs](https://code.claude.com/docs), 2026-10-03 확인). 책과 달라진 부분은 **📌 현재 공식 문서 기준** 으로 표시했다.
>
> 관련 노트: [1장. 시작하기](./01-getting-started.md) (컨텍스트·compaction 개념) · [3장. 에이전트 스킬](./03-agent-skills.md) · [4장. 서브에이전트](./04-subagents.md)

---

## 2-1. 기본 작업 방식

### 탐색 → 계획 → 구현 → 커밋

**무엇** — 공식 문서가 권장하는 4단계 워크플로. 탐색·계획을 구현과 분리해서 "엉뚱한 문제를 푸는 코드"를 막는다.

**언제** — 접근 방법이 불확실할 때, 여러 파일을 바꿀 때, 낯선 코드를 수정할 때. 반대로 **diff를 한 문장으로 설명할 수 있으면**(오타, 로그 한 줄, 변수명 변경) 계획 없이 바로 시킨다.

| 단계 | 방법 | 포인트 |
|---|---|---|
| 1. 탐색 | Plan 모드 진입: `Shift+Tab` (상태바 `⏸ plan mode on`), `claude --permission-mode plan`, 또는 프롬프트 앞에 `/plan` | 파일 읽기·탐색 명령만 실행, 소스는 수정하지 않음 |
| 2. 계획 | "무엇을 바꿔야 하는지, 흐름은 어떤지 계획 세워줘" | `Ctrl+G` 로 계획을 편집기에서 직접 수정 가능 |
| 3. 구현 | 계획 승인 (또는 `Shift+Tab` 으로 Plan 모드 해제) | 승인 옵션: auto 모드로 진행 / 편집마다 수동 승인 / 계속 계획 |
| 4. 커밋 | "설명적인 메시지로 커밋하고 PR 열어줘" | Git 통합 워크플로 참고 |

**예시 (Java/JSP 프로젝트)**

```text
# 1. 탐색 (plan mode)
WEB-INF/jsp/member 화면과 MemberController, MemberService, MemberDAO를 읽고
로그인 요청이 JSP → Controller → Service → DAO로 어떻게 흐르는지 설명해줘.

# 2. 계획
로그인 5회 실패 시 계정을 잠그려면 어떤 파일을 바꿔야 해? 계획 세워줘.

# 3. 구현
계획대로 구현하고 MemberServiceTest에 잠금 테스트를 추가해.
테스트를 실행해서 실패하면 고쳐줘.

# 4. 커밋
변경 내용을 설명하는 메시지로 커밋해줘.
```

🔗 [Best practices – Explore first, then plan, then code](https://code.claude.com/docs/en/best-practices#explore-first-then-plan-then-code) · [Plan mode](https://code.claude.com/docs/en/permission-modes#analyze-before-you-edit-with-plan-mode)

### 탐색과 분석

**무엇** — 시니어 엔지니어에게 묻듯 Claude에게 질문하며 코드베이스를 파악하는 것. 특별한 프롬프트 기법은 필요 없다.

**언제** — 새 프로젝트 온보딩, 기능 위치 찾기, 실행 흐름 추적, "왜 이렇게 짰지?" 같은 이력 조사.

**방법**

- **넓게 → 좁게**: "이 코드베이스 개요" → "주요 데이터 모델은?" → "인증은 어떻게 처리돼?"
- **프로젝트 용어 사용 + 위치 지정**: 파일명·패키지명·기존 예시 패턴을 짚어준다.
- **`@` 참조**: `@파일` 은 내용을 바로 포함, `@디렉터리` 는 파일 목록만 포함.
- **git 히스토리 활용**: "OrderService의 git 히스토리를 보고 API가 왜 이렇게 됐는지 요약해줘"
- **서브에이전트에 위임**: "use a subagent to investigate …" → 서브에이전트가 별도 컨텍스트에서 파일을 읽고 요약만 돌려준다.
- **code intelligence 플러그인**: 언어별 플러그인을 설치하면 정의 이동·참조 찾기가 정확해진다.

```text
@src/main/java/com/acme/order/OrderService.java 의 주문 취소 로직이
어떤 테이블을 수정하는지 DAO와 MyBatis 매퍼까지 추적해줘.
```

> ⚠️ 범위 없이 "조사해줘"라고 하면 수백 개 파일을 읽으며 컨텍스트를 소모한다(공식 문서의 *infinite exploration* 실패 패턴). 범위를 좁히거나 서브에이전트를 쓰자. 컨텍스트 개념은 [1장](./01-getting-started.md) 참고.

🔗 [Common workflows – Understand new codebases](https://code.claude.com/docs/en/common-workflows#understand-new-codebases) · [Use subagents for investigation](https://code.claude.com/docs/en/best-practices#use-subagents-for-investigation)

### 점진적 개발 전략

**무엇** — 작게 나눠 진행하고, 매 단계 Claude가 직접 확인할 수 있는 검증 신호를 주고, 빗나가면 즉시 되돌리는 방식.

**언제** — 거의 항상. 특히 리팩터링·레거시 수정.

**1) 검증 수단을 준다** — Claude는 "끝난 것처럼 보이면" 멈춘다. 테스트, 빌드 종료 코드, 린터처럼 pass/fail 신호를 주면 스스로 고치고 다시 확인하는 루프가 닫힌다. 성공했다는 말 대신 **증거**(테스트 출력, 실행한 명령)를 보여달라고 하자.

```text
빌드가 이 에러로 실패해: [에러 붙여넣기]
고치고 mvn test가 통과하는지 확인해. 에러를 억제하지 말고 근본 원인을 해결해.
```

**2) 작은 단위로** — "작고 테스트 가능한 단위로 리팩터링"하고, 단위마다 커밋한다. 큰 기능은 Claude에게 인터뷰(AskUserQuestion)를 시켜 `SPEC.md` 를 만든 뒤 **새 세션**에서 구현하는 방법도 있다.

**3) 빨리 방향을 바꾼다**

| 명령 | 동작 |
|---|---|
| `Esc` | 작업 중단 (컨텍스트는 유지, 바로 방향 지시) |
| `Esc` 두 번 / `/rewind` | 체크포인트로 대화·코드·둘 다 복원, 또는 특정 지점부터/까지 요약 |
| `/clear` | 관련 없는 작업 사이에 컨텍스트 초기화 (이전 대화는 `/resume` 으로 복귀 가능) |
| `/compact 지시문` | 대화를 요약해 공간 확보 (개념은 1장) |

규칙: **같은 문제를 두 번 넘게 교정했다면** `/clear` 후 배운 점을 반영한 더 구체적인 프롬프트로 새로 시작한다.

> ⚠️ 체크포인트는 Claude의 **파일 편집 도구로 바꾼 내용만** 추적한다. Bash 명령이나 외부 프로세스가 바꾼 파일은 복원되지 않으므로 git을 대체하지 못한다. 단계마다 커밋하자.

🔗 [Give Claude a way to verify its work](https://code.claude.com/docs/en/best-practices#give-claude-a-way-to-verify-its-work) · [Course-correct early and often](https://code.claude.com/docs/en/best-practices#course-correct-early-and-often) · [Checkpointing](https://code.claude.com/docs/en/checkpointing)

### Git 통합 워크플로

**무엇** — 변경 요약, 커밋 메시지, PR 작성, 히스토리 분석, 병렬 브랜치 작업을 Claude와 함께 하는 것.

**언제** — 작업 단위가 끝났을 때, PR을 올릴 때, 여러 작업을 동시에 진행할 때.

**커밋·PR**

- "인증 모듈 변경 사항 요약해줘" → "create a pr" → "PR 설명에 보안 관련 맥락을 보강해줘"
- GitHub를 쓴다면 `gh` CLI 설치 권장 (없으면 API 요청이 rate limit에 걸리기 쉽다).
- Claude가 `gh pr create` 로 만든 PR은 세션과 연결된다 → `claude --from-pr 123` 으로 그 세션을 다시 찾을 수 있다.
- 제출 전 생성된 PR을 직접 검토하고 위험 요소를 짚어달라고 요청하자.

**병렬 작업 – worktree**

```powershell
claude --worktree feature-login-lock   # 또는 -w
```

`.claude/worktrees/이름` 에 별도 체크아웃·브랜치를 만들어 실행한다. 다른 터미널에서 다른 이름으로 실행하면 편집이 충돌하지 않는다. 저장소에 **커밋이 최소 1개** 있어야 한다.

**커밋 서명 문구 – `attribution`**

회사 규정상 커밋/PR에 Claude 서명(Co-authored-by)을 넣지 않으려면:

```json
{
  "attribution": {
    "commit": false,
    "pr": false
  }
}
```

> 📌 **현재 공식 문서 기준**: 예전 `includeCoAuthoredBy` 설정은 deprecated. `attribution` 을 사용한다.

🔗 [Create pull requests](https://code.claude.com/docs/en/common-workflows#create-pull-requests) · [Worktrees](https://code.claude.com/docs/en/worktrees) · [attribution](https://code.claude.com/docs/en/settings-reference#attribution)

### 비대화형 모드와 세션 재개

**무엇** — `claude -p "프롬프트"` 는 대화 화면 없이 실행하고 결과를 출력한 뒤 종료한다. 실행 결과는 세션으로 저장되어 나중에 이어갈 수 있다.

**언제** — 스크립트, CI, pre-commit 훅, 여러 파일 일괄 처리, 이전 세션에 후속 질문 보내기.

> 📌 **현재 공식 문서 기준**: "헤드리스 모드" 문서는 *Run Claude Code programmatically*(Agent SDK의 CLI 사용)로 정리됐다. 스크립트·CI에는 `--bare` 를 권장한다. hooks·skills·플러그인·MCP·`CLAUDE.md` 자동 로드를 건너뛰어 어느 PC에서나 같은 결과가 나오며, 앞으로 `-p` 의 기본값이 될 예정이다. 단 `--bare` 는 구독 로그인 대신 `ANTHROPIC_API_KEY` 가 필요하다.

**세션 재개 명령**

| 명령 | 동작 |
|---|---|
| `claude --continue` (`-c`) | 현재 디렉터리의 가장 최근 대화 이어가기 |
| `claude --resume` (`-r`) | 세션 선택 목록 열기 (`Ctrl+A` 전체 프로젝트, `Ctrl+B` 현재 브랜치) |
| `claude --resume 이름또는ID` | 특정 세션 바로 재개 (ID는 다른 디렉터리에서도 찾음) |
| `claude -n login-lock` / `/rename` | 세션 이름 붙이기 → 작업별 "브랜치"처럼 관리 |
| `/resume` | 세션 안에서 다른 대화로 전환 |
| `/branch` / `--fork-session` | 원본은 그대로 두고 복사본으로 분기해서 다른 접근 시도 |

**`-p` 에서 이어가기**

```bash
claude -p "이 코드베이스의 성능 문제를 리뷰해줘"
claude -p "이제 DB 쿼리에 집중해줘" --continue
```

**세션 ID를 잡아서 특정 세션 이어가기 (PowerShell)** — 공식 예시는 bash + `jq` 이고, 아래는 PowerShell의 `ConvertFrom-Json` 으로 바꾼 것이다.

```powershell
$r = claude -p "주문 모듈 리뷰 시작" --output-format json | ConvertFrom-Json
claude -p "그 리뷰 이어서 결제 쪽도 봐줘" --resume $r.session_id
```

**알아둘 점**

- `-p` 로 만든 세션은 대화형 `claude --continue` 와 세션 선택 목록에 나오지 않는다 → 세션 ID로 `--resume` 한다. (`claude -p --continue` 는 포함)
- 재개할 때 `--settings`, `--add-dir`, `--mcp-config` 같은 실행 플래그는 복원되지 않으므로 다시 넘긴다. settings.json 파일은 매번 다시 읽는다.
- 대화 기록 위치: `~/.claude/projects/프로젝트/세션ID.jsonl` (Windows: `%USERPROFILE%\.claude\projects\…`), 기본 **30일** 보관 → `cleanupPeriodDays` 로 조정. 저장하지 않으려면 `-p` 에 `--no-session-persistence`. 회사 소스 대화가 로컬에 남는 것이 부담되면 보관 기간을 줄이자.
- 자주 쓰는 `-p` 옵션: `--max-turns`, `--max-budget-usd`, `--allowedTools`, `--permission-mode`, `--append-system-prompt`

🔗 [Run Claude Code programmatically](https://code.claude.com/docs/en/headless) · [Manage sessions](https://code.claude.com/docs/en/sessions) · [CLI reference](https://code.claude.com/docs/en/cli-reference)

### 파이프와 Claude Code

**무엇** — `-p` 는 stdin을 읽으므로 다른 명령의 출력을 파이프로 넣고, 결과를 파일이나 다른 명령으로 보낼 수 있다. 일반 Unix 도구처럼 쓰는 것이다.

**언제** — 로그 분석, diff 리뷰, 커밋 요약, 빌드 스크립트에 "AI 린터" 추가, 결과를 JSON으로 받아 후처리.

```bash
git log --oneline -20 | claude -p "최근 커밋을 요약해줘"
cat build-error.txt | claude -p "이 빌드 에러의 근본 원인을 간단히 설명해줘" > output.txt
git diff main | claude -p "이 diff에서 오타를 찾아 파일:라인 형식으로만 보고해줘"
```

```powershell
# Windows PowerShell 예시
Get-Content .\logs\catalina.out -Tail 200 | claude -p "이 Tomcat 로그의 에러 원인을 정리해줘"
git diff --staged | claude -p "staged 변경으로 커밋 메시지 초안을 써줘"
```

**출력 형식**

| 옵션 | 결과 |
|---|---|
| `--output-format text` | 기본값, 일반 텍스트 |
| `--output-format json` | JSON 하나: `result`, `session_id`, 사용량, 예상 비용 |
| `--output-format stream-json --verbose` | 이벤트를 한 줄에 하나씩 실시간 출력 |
| `--json-schema '스키마'` | 스키마에 맞춘 결과를 `structured_output` 필드로 |

**알아둘 점**

- 파이프 입력은 **최대 10MB**. 넘으면 파일로 저장하고 프롬프트에 경로를 넘긴다.
- diff를 파이프로 넣으면 Claude가 Bash로 git을 실행할 필요가 없다 → 권한을 최소화할 수 있다.
- 성공 시 종료 코드 0, 실패 시 0이 아님 → 스크립트에서 분기할 수 있다.

> 💡 **PowerShell 일반 팁 (Claude Code 문서 내용 아님)**: Windows PowerShell 5.1은 외부 프로그램으로 파이프할 때 `$OutputEncoding` 기본값(ASCII) 때문에 한글이 `?` 로 깨질 수 있다. 세션에서 `$OutputEncoding = [System.Text.Encoding]::UTF8` 을 먼저 실행하거나 PowerShell 7을 사용하자.

🔗 [Pipe data through Claude](https://code.claude.com/docs/en/headless#pipe-data-through-claude) · [Get structured output](https://code.claude.com/docs/en/headless#get-structured-output) · [Pipe Claude into scripts](https://code.claude.com/docs/en/common-workflows#pipe-claude-into-scripts)

---

## 2-2. 설정 관리

### settings.json과 설정 우선순위

**무엇** — 권한, 환경 변수, hooks, 모델 같은 Claude Code 동작을 JSON 파일로 설정한다. 범위별로 파일 4개와 조직용 managed 설정이 있다.

| 범위 | 파일 | 적용 대상 | 용도 |
|---|---|---|---|
| User | `~/.claude/settings.json` (Windows: `%USERPROFILE%\.claude\settings.json`) | 내 PC의 모든 프로젝트 | 개인 취향, 나만의 권한 규칙 |
| Project (공유) | `.claude/settings.json` | 저장소를 쓰는 모두 (git 커밋) | 팀 권한, hooks, 플러그인, 필요한 env |
| Project local | `.claude/settings.local.json` | 나, 이 프로젝트만 (git 제외) | 개인 예외, 공유 전 테스트 |
| Managed | `managed-settings.json`, MDM, 서버 관리 설정 | 조직 전체 | 보안·컴플라이언스 정책 (사용자가 덮어쓸 수 없음) |

**우선순위 (높음 → 낮음)**: ① Managed → ② 명령줄 인자 (`--settings`, `--model` 등, 그 세션만) → ③ Project local → ④ Project → ⑤ User

- 같은 키는 높은 쪽 값이 이긴다. 단 **배열 키(`permissions.allow` 등)는 덮어쓰지 않고 합쳐진다**.
- 권한 규칙은 이와 별도로 **어느 범위든 deny가 있으면 allow로 풀 수 없다** (아래 도구 및 권한 설정 참고).
- 권한 확인 창에서 "Yes, and don't ask again"을 고르면 `.claude/settings.local.json` 에 allow 규칙이 자동 저장된다. Claude Code가 이 파일을 처음 쓸 때 전역 git excludes에 추가하므로 커밋되지 않는다. 직접 만든 파일이라면 `.gitignore` 에 직접 추가해야 한다.
- 공유 `.claude/settings.json` 은 세션을 시작한 디렉터리에서 읽는다 → **저장소 루트에서 `claude` 를 실행**하자.
- 대부분의 수정은 재시작 없이 바로 반영된다 (`model` 등 일부 키는 다음 세션부터).

> 📌 **현재 공식 문서 기준**: git 저장소 하위 폴더에서 시작해도 local 파일은 보통 저장소 루트에 둔다. 하지만 **Windows에서는** 루트로 옮기지 않고 `.claude/settings.json` 과 같은 위치에 둔다.

**확인 방법** — `/status` 의 `Setting sources` 줄에서 실제로 로드된 파일을 확인하고, 거부된 항목은 터미널에서 `claude doctor` 로 확인한다.

🔗 [Settings files and precedence](https://code.claude.com/docs/en/settings)

### 주요 설정 옵션

**무엇** — settings.json에서 자주 쓰는 키들. 전체 목록은 settings-reference에 있다.

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "opus",
  "permissions": {
    "defaultMode": "default",
    "allow": ["Bash(git diff *)"],
    "deny": ["Read(./.env)"]
  },
  "env": {
    "BASH_DEFAULT_TIMEOUT_MS": "300000"
  },
  "cleanupPeriodDays": 14
}
```

| 키 | 설명 |
|---|---|
| `permissions` | allow / ask / deny 규칙, `defaultMode`, `additionalDirectories` 등 |
| `env` | 모든 세션과 하위 프로세스에 적용할 환경 변수 |
| `model` | 새 세션 기본 모델 (별칭 또는 전체 ID) |
| `hooks` | 도구 실행 전후 등 특정 시점에 내 명령 실행 |
| `attribution` | 커밋/PR 서명 문구 변경·숨김 |
| `cleanupPeriodDays` | 대화 기록 보관 일수 (기본 30) |
| `sandbox` | 셸 명령의 파일·네트워크 접근을 OS 수준에서 격리 (macOS·Linux·WSL2) |
| `language` | Claude 응답 언어 |
| `respectGitignore` | gitignore된 파일을 `@` 파일 선택기에서 제외 |
| `defaultShell` | `!` 로 직접 입력하는 명령을 Bash로 실행할지 PowerShell로 실행할지 |
| `autoUpdatesChannel` | `stable` 채널로 업데이트 |

**설정을 바꾸는 3가지 방법**

- `/config` 메뉴: 테마 같은 일부 개인 옵션만 있다 (모든 키가 있는 것은 아님). 하나만 바꿀 때는 `/config key=value`.
- 파일 직접 편집: **엄격한 JSON** 이라 주석(`//`)이나 끝의 쉼표가 있으면 Settings Error가 난다. `$schema` 줄을 넣으면 JSON 스키마를 지원하는 편집기에서 자동완성·검증이 된다.
- 한 세션만: `claude --settings 파일또는JSON`. PowerShell에서는 인라인 JSON 따옴표 처리가 번거로우니 `claude --settings .\try-settings.json` 처럼 파일로 넘기는 편이 편하다.

🔗 [All settings](https://code.claude.com/docs/en/settings-reference) · [Change a setting](https://code.claude.com/docs/en/settings#change-a-setting)

### 프로젝트별 설정

**무엇** — 프로젝트마다 다른 권한·환경을 두되, 공유할 것과 개인 것을 파일로 나누는 전략.

| 무엇을 | 어디에 | 예 |
|---|---|---|
| 팀이 같이 써야 하는 것 | `.claude/settings.json` (커밋) | 빌드·테스트 명령 허용, 민감 파일 deny, push 확인, 팀 hooks |
| 나만의 예외 | `.claude/settings.local.json` | 내 로컬 경로, 개인적으로 더 허용하는 명령 |
| 모든 프로젝트 공통 | `~/.claude/settings.json` | 개인 기본 deny (`~/.ssh` 등), 기본 권한 모드 |

**예시 – Maven 기반 Java/JSP 프로젝트 `.claude/settings.json`**

```json
{
  "permissions": {
    "allow": [
      "Bash(mvn compile *)",
      "Bash(mvn test *)",
      "Bash(git commit *)",
      "PowerShell(mvn compile *)",
      "PowerShell(mvn test *)",
      "PowerShell(git commit *)"
    ],
    "ask": [
      "Bash(git push *)",
      "PowerShell(git push *)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(**/application-prod*.properties)",
      "Read(**/*.jks)"
    ],
    "defaultMode": "plan"
  }
}
```

- **Windows**: Git Bash가 설치돼 있어도 `claude.ai`·Console 계정에서는 PowerShell 도구가 기본으로 켜지고 **기본 셸**로 쓰인다. 그래서 셸 명령 규칙은 `Bash(...)` 와 `PowerShell(...)` 을 **둘 다** 적어두는 것이 안전하다.
- 규칙은 **쓰인 그대로** 매칭된다: `Bash(mvn test *)` 는 `mvn test -Dtest=X` 에는 맞지만 `mvn -q test` 에는 맞지 않는다.
- 공유 파일의 **allow 규칙은 각자가 폴더를 trust한 뒤에** 적용된다. deny·ask는 바로 적용된다.
- `defaultMode` 를 `plan` 으로 두면 레거시 프로젝트를 "분석부터" 시작하게 할 수 있다. 단 프로젝트 파일에 `auto` 나 `bypassPermissions` 를 넣어도 적용되지 않는다.
- **다른 폴더 참조** (예: 공통 라이브러리): `claude --add-dir ..\common-lib`, 세션 중 `/add-dir`, 영구 설정은 `permissions.additionalDirectories`. 이 방식은 파일 접근만 허용하고, 그 폴더의 `.claude` 설정은 대부분 읽지 않는다. 네트워크 경로(`\\server\share`)는 추가할 수 없으므로 드라이브 문자로 매핑해서 쓴다.

🔗 [Working directories](https://code.claude.com/docs/en/permissions#working-directories) · [Start in a different permission mode](https://code.claude.com/docs/en/permission-modes#start-in-a-different-mode) · [PowerShell tool](https://code.claude.com/docs/en/tools-reference#powershell-tool)

### 환경 변수

**무엇** — 타임아웃, 프록시, 텔레메트리, 셸 경로 같은 Claude Code 동작을 바꾸는 변수.

**언제** — 사내망(프록시·인증서), 오래 걸리는 빌드, Windows 셸 경로 문제, 개인정보 관련 트래픽 차단.

**설정 방법**

```powershell
# PowerShell (현재 창에서만 유효)
$env:BASH_DEFAULT_TIMEOUT_MS = "300000"
claude
```

```bat
REM CMD
set BASH_DEFAULT_TIMEOUT_MS=300000
claude
```

```json
{
  "env": {
    "BASH_DEFAULT_TIMEOUT_MS": "300000"
  }
}
```

**우선순위**

- 셸과 settings의 `env` 에 같은 변수가 있으면 **대부분 settings 값이 적용**된다. 예외: `ANTHROPIC_MODEL`, `CLAUDE_CODE_AUTO_CONNECT_IDE` 는 셸 변수가 우선한다.
- settings 파일끼리는 managed → project/local → user 순.
- 상위에서 설정돼 지울 수 없는 변수는 `env` 에 `""` (빈 문자열)로 덮어쓴다.
- → 실무에서는 팀 공통 값은 `.claude/settings.json` 의 `env`, 개인 값은 user settings의 `env` 에 두면 매번 셸에서 설정할 필요가 없다.

**실무에서 쓸 만한 변수**

| 변수 | 용도 |
|---|---|
| `CLAUDE_CODE_GIT_BASH_PATH` | Windows: Git Bash가 PATH에 없을 때 `bash.exe` 경로 지정 |
| `CLAUDE_CODE_USE_POWERSHELL_TOOL` | PowerShell 도구 켜기(1) / 끄기(0) |
| `BASH_DEFAULT_TIMEOUT_MS` / `BASH_MAX_TIMEOUT_MS` | 셸 명령 기본 2분 / 최대 10분. Maven 전체 빌드처럼 오래 걸리면 늘린다 (PowerShell 도구도 같은 값 사용) |
| `HTTPS_PROXY` / `HTTP_PROXY` | 사내 프록시 |
| `NODE_EXTRA_CA_CERTS` / `CLAUDE_CODE_CERT_STORE` | 사내 CA 인증서 추가 / 인증서 출처 (`bundled,system` 이면 OS 신뢰 저장소도 사용) |
| `API_TIMEOUT_MS` | 느린 네트워크·프록시에서 API 타임아웃 연장 |
| `DISABLE_TELEMETRY` | 텔레메트리 수집 끄기 |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | 자동 업데이트·텔레메트리·오류 보고 등 비필수 트래픽 차단. ⚠ `0` 으로 설정해도 차단되므로 해제하려면 변수를 삭제해야 한다 |
| `CLAUDE_CONFIG_DIR` | `~/.claude` 대신 다른 설정 폴더 사용 |
| `ANTHROPIC_API_KEY` | 설정돼 있으면 구독 로그인 대신 이 키로 과금된다 (의도치 않게 남아 있지 않은지 주의) |

> ⚠️ API 키·토큰 같은 비밀 값은 커밋되는 `.claude/settings.json` 의 `env` 에 넣지 말자. 개인 user settings나 OS 환경 변수에 둔다.

🔗 [Environment variables](https://code.claude.com/docs/en/env-vars)

### 팀 표준 설정

**무엇** — 저장소에 `.claude/settings.json` 을 커밋해서 팀 전체가 같은 권한·hooks·플러그인을 쓰게 하는 것. 개인 예외는 각자 local 파일에 둔다.

**언제** — 여러 명이 같은 저장소에서 Claude Code를 쓸 때, 신규 입사자 온보딩.

**공식 예시 (일부 발췌)**

```json
{
  "permissions": {
    "allow": ["Bash(npm run *)"],
    "ask": ["Bash(git push *)"],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh"
          }
        ]
      }
    ]
  },
  "sandbox": {
    "enabled": true,
    "network": {
      "allowedDomains": ["registry.npmjs.org"]
    }
  }
}
```

**커밋 전 체크리스트**

- allow 규칙과 `extraKnownMarketplaces` 는 각 팀원이 폴더를 trust한 뒤에 적용된다. deny·ask는 trust 여부와 관계없이 바로 적용된다.
- 일부 키는 저장소 파일에서 무시된다 (User/Managed 전용 키, `defaultMode` 의 `auto`·`bypassPermissions`, 텔레메트리 export 변수).
- hooks는 저장소 안의 스크립트를 가리킨다. Windows 팀원이 있다면 PowerShell 스크립트로 작성하고 hook에 `"shell": "powershell"` 을 지정할 수 있다. 셸 명령을 검사하는 hook의 matcher는 `Bash` 만이 아니라 `Bash|PowerShell` 로 둔다.
- 규칙은 쓰인 형태만 매칭한다: `Bash(git push *)` 는 `git -C . push` 를 잡지 못한다.
- 빌드 명령·코딩 규칙은 짧은 `CLAUDE.md` 로 함께 커밋한다.

**조직 단위 강제 – managed settings** — 관리자가 배포하며 사용자·프로젝트 설정으로 덮어쓸 수 없다. 예: deny 규칙, `disableBypassPermissionsMode`, `allowManagedPermissionRulesOnly` (managed 권한 규칙만 인정), `sandbox.failIfUnavailable`. 내게 적용 중인 정책은 `/status` 에서 확인한다.

🔗 [A team's shared settings](https://code.claude.com/docs/en/settings-example#a-teams-shared-settings) · [Deploy managed settings](https://code.claude.com/docs/en/managed-settings) · [PowerShell tool](https://code.claude.com/docs/en/tools-reference#powershell-tool)

### 민감한 파일 보호

**무엇** — Claude가 비밀 정보가 담긴 파일(.env, 키스토어, 운영 DB 설정 등)을 읽거나 수정하지 못하게 막는 것.

**언제** — 회사 소스를 다룰 때는 항상. 처음 프로젝트에 Claude Code를 붙일 때 가장 먼저 설정한다.

> 📌 **현재 공식 문서 기준**: `.claudeignore` 파일은 **효과가 없다** (공식 문서에 명시). 그 내용을 `permissions.deny` 의 `Read(...)` 규칙으로 옮겨야 한다. `Read` deny는 같은 경로의 Edit/Write(새 파일 생성 포함)도 막는다. NotebookEdit는 막지 않으므로 필요하면 `Edit(...)` deny를 추가한다.

**경로 패턴 (gitignore 문법)**

| 패턴 | 의미 | 예 |
|---|---|---|
| `//경로` | 파일 시스템 루트 기준 절대 경로 | `Read(//c/Users/me/keys/**)` |
| `~/경로` | 홈 디렉터리 기준 | `Read(~/.ssh/**)` |
| `/경로` | 설정 파일 기준 (프로젝트 설정이면 프로젝트 루트, **user 설정이면 `~/.claude`**) | `Edit(/src/**)` |
| `경로` / `./경로` | 현재 디렉터리 기준 (파일명만 쓰면 모든 깊이) | `Read(.env)` = `Read(**/.env)` |

- **Windows 경로**는 POSIX 형식으로 바뀐 뒤 매칭된다: `C:\Users\alice` → `/c/Users/alice`. C 드라이브 전체의 .env는 `//c/**/.env`, 모든 드라이브는 `//**/.env`.
- ⚠ 함정: `/Users/alice/file` 은 절대 경로가 아니다 (`//` 가 필요). user 설정의 `Read(/secrets/**)` 는 프로젝트가 아니라 `~/.claude/secrets` 를 막는다.

**예시 – Java/JSP 프로젝트에서 막을 후보**

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(**/application-prod*.properties)",
      "Read(**/application-prod*.yml)",
      "Read(**/jdbc*.properties)",
      "Read(**/*.jks)",
      "Read(**/*.p12)",
      "Read(**/*.pem)",
      "Read(~/.ssh/**)",
      "Read(~/.m2/settings.xml)"
    ]
  }
}
```

(`~/.m2/settings.xml` 에는 사내 저장소 계정 정보가 들어 있는 경우가 많다.)

**한계와 보완 – 꼭 알아둘 것**

- `Read`/`Edit` deny가 적용되는 범위: Claude의 파일 도구, `cat`·`head`·`tail`·`sed`·`tee` 처럼 Claude Code가 인식하는 파일 명령, 리다이렉션 대상(`> file`, `< file`).
- **적용되지 않는 범위**: `grep -r pattern .` 처럼 파일명을 직접 쓰지 않는 명령, Python·Node 스크립트처럼 간접적으로 파일을 여는 프로세스.
- 완전히 막으려면 **sandbox(OS 수준 격리)** 가 필요하다. 하지만 sandbox는 macOS·Linux·**WSL2만 지원**하고 **네이티브 Windows에서는 동작하지 않는다**. Windows에서는 다음 방법으로 보완한다.
  - WSL2 안에서 Claude Code 실행 후 sandbox 사용
  - 비밀 파일을 작업 디렉터리 밖에 두고 `permissions.blockReadsOutsideWorkingDirectories` 켜기 (모든 권한 모드에서 파일 도구가 작업 디렉터리 밖 읽기를 거부)
  - PreToolUse hook으로 명령 내용을 검사해 차단
- **신뢰하지 않은 저장소에서 `claude -p`**: `-p` 는 trust 확인 창이 없어서 그 저장소의 hooks와 `.mcp.json` 서버를 그대로 실행한다 → `--bare` 또는 `--setting-sources user`, `--settings '{"disableAllHooks": true}'` 를 사용한다.
- 프롬프트나 `CLAUDE.md`의 "이 파일 읽지 마"는 **강제력이 없다**. 강제하는 것은 권한 규칙·hooks·sandbox다.

🔗 [Read and Edit rules](https://code.claude.com/docs/en/permissions#read-and-edit) · [Sandboxing](https://code.claude.com/docs/en/sandboxing) · [What runs before you trust a folder](https://code.claude.com/docs/en/permissions#what-runs-before-you-trust-a-folder)

### 도구 및 권한 설정

**무엇** — 어떤 도구를 묻지 않고 쓸지(allow), 매번 물을지(ask), 절대 못 쓰게 할지(deny)를 규칙으로 정하고, 세션 전체의 기본 태도는 권한 모드로 정한다.

**기본 승인 정책 (Manual 모드)**

| 도구 유형 | 승인 필요? | "다시 묻지 않기" 선택 시 |
|---|---|---|
| 읽기 (Read, Grep) | 작업 디렉터리 안이면 불필요 | - |
| 셸 명령 (Bash, PowerShell) | 필요 (`ls`, `cat`, `git status` 같은 내장 읽기 전용 명령은 제외) | 저장소·명령별로 영구 |
| 파일 수정 (Edit, Write) | 필요 | 세션이 끝날 때까지 |
| WebFetch | 필요 (일부 문서 도메인 제외) | 저장소·도메인별로 영구 |
| WebSearch | 필요 | 저장소별로 영구 |

**평가 순서: deny → ask → allow** — 이 순서로 처음 맞는 규칙이 결정한다. 규칙이 얼마나 구체적인지는 상관없다. 넓은 deny `Bash(aws *)` 는 좁은 allow `Bash(aws s3 ls)` 를 이긴다. 설정 파일 범위와도 상관없이 **어느 파일의 deny든** 다른 파일의 allow보다 우선한다. `/permissions` 에서 현재 규칙과 각 규칙의 출처 파일을 확인·추가할 수 있다.

**규칙 문법**

| 규칙 | 의미 |
|---|---|
| `Bash`, `Read`, `WebFetch` | 도구 전체 (deny에 쓰면 도구 자체가 Claude에게서 사라짐) |
| `Bash(npm run build)` | 정확히 그 명령만 |
| `Bash(git log *)` | `git log` 로 시작하는 명령 (`git log` 자체 포함) |
| `Bash(git *)` | 모든 git 명령 → 너무 넓다 |
| `Bash(ls*)` (공백 없음) | `ls` 와 `lsof` 둘 다 |
| `PowerShell(Get-ChildItem *)` | 별칭(`gci`, `ls`, `dir`)도 매칭, 대소문자 무시 |
| `Read(./.env)`, `Edit(/src/**)` | 파일 경로 (gitignore 문법) |
| `WebFetch(domain:docs.spring.io)` | 도메인 단위 |
| `mcp__notion`, `mcp__notion__*` | MCP 서버의 모든 도구 |
| `Agent(Explore)` | 특정 서브에이전트 |

> 📌 **현재 공식 문서 기준**: 끝에 붙이는 `:*` 형식(`Bash(ls:*)`)도 동작하지만, 지금은 **공백 + `*`** 형식(`Bash(ls *)`)이 기본이다. `:*` 는 패턴 끝에서만 인식된다. 서브에이전트 규칙의 도구 이름은 `Agent` 다.

**Bash 규칙 주의점**

- `*` 는 하위 명령 뒤에 둔다: `Bash(git log *)` 는 괜찮지만 `Bash(git * main)` 은 시작할 때 경고가 뜬다.
- 복합 명령(`&&`, `||`, `;`, `|`)은 **각 부분이 모두** 규칙에 맞아야 허용된다. deny·ask는 어느 한 부분만 맞아도 적용된다.
- 규칙은 쓰인 형태만 매칭하므로 **보안 경계가 아니다**: `Bash(rm *)` deny는 `/bin/rm …` 이나 `bash -c 'rm …'` 을 막지 못한다.
- Bash 규칙으로 URL을 제한하는 방식(`Bash(curl http://github.com/ *)`)은 깨지기 쉽다 → `curl`/`wget` 은 deny하고 `WebFetch(domain:…)` 만 allow하는 편이 낫다.

**권한 모드 6가지**

| 모드 | 묻지 않고 실행하는 것 | 적합한 상황 |
|---|---|---|
| Manual (`default`) | 읽기만 | 민감한 작업, 처음 보는 코드 |
| `acceptEdits` | 읽기 + 파일 편집 + `mkdir`·`mv`·`cp` 등 (작업 디렉터리 안) | 편집은 나중에 `git diff` 로 몰아서 검토할 때 |
| `plan` | 읽기 (+ 탐색 명령) | 수정 전 분석 |
| `auto` | 분류기 모델이 검토한 뒤 대부분 실행 | 긴 작업, 확인 창이 너무 많을 때 |
| `dontAsk` | 미리 허용한 것만 실행, 나머지는 자동 거부 | CI, 스크립트 |
| `bypassPermissions` | 거의 전부 | 격리된 컨테이너·VM에서만 |

전환: `Shift+Tab` 으로 순환 (auto → Manual → acceptEdits → plan), 시작할 때 `--permission-mode`, 기본값은 `permissions.defaultMode`. **deny 규칙은 bypassPermissions를 포함한 모든 모드에서 적용된다.**

> 📌 **현재 공식 문서 기준**: v2.1.283부터 대화형 터미널·VS Code 세션은 **auto 모드로 시작**한다 (매번 묻는 모드는 이제 "Manual"로 표시). `claude -p` 의 시작 모드는 환경에 따라 다르므로 `--permission-mode` 를 명시하라고 안내한다. 공식 문서도 auto 모드가 "안전을 보장하지 않는다"고 경고한다. `--enable-auto-mode` 플래그는 제거됐다.
>
> → 회사 소스를 다루며 매번 직접 확인하고 싶다면 user settings에 `"defaultMode": "default"` 를 명시하자.

**예시 – 민감한 회사 소스용 개인 기본 설정 (`~/.claude/settings.json`)** — 공식 권장안이 아니라 공식 문서의 규칙으로 구성한 예시다.

```json
{
  "permissions": {
    "defaultMode": "default",
    "disableBypassPermissionsMode": "disable",
    "ask": [
      "Bash(git push *)",
      "PowerShell(git push *)"
    ],
    "deny": [
      "Read(~/.ssh/**)",
      "Read(~/.m2/settings.xml)",
      "Bash(curl *)",
      "Bash(wget *)",
      "PowerShell(Invoke-WebRequest *)"
    ]
  }
}
```

🔗 [Configure permissions](https://code.claude.com/docs/en/permissions) · [Choose a permission mode](https://code.claude.com/docs/en/permission-modes) · [Tools reference](https://code.claude.com/docs/en/tools-reference)

### 권한/도구 설정 방식 비교

| 방식 | 설정 위치 | 강제력 | 적합한 용도 |
|---|---|---|---|
| `CLAUDE.md` 지시 | `CLAUDE.md` | ❌ 권고만 (Claude의 행동을 유도할 뿐) | 코딩 규칙, 작업 습관 |
| permissions 규칙 (allow/ask/deny) | settings.json, `/permissions` | ✅ Claude Code가 강제 (단 명령 문자열을 쓰인 그대로 매칭) | 기본 수단: 허용 명령, 민감 파일, push 확인 |
| 권한 모드 | `Shift+Tab`, `--permission-mode`, `defaultMode` | ✅ 세션의 기본 태도 | 분석 = plan, 편집 반복 = acceptEdits, CI = dontAsk |
| `--allowedTools` | CLI | 이 세션에서 **묻지 않고** 실행할 도구 추가 | `-p` 일괄 작업 |
| `--disallowedTools` | CLI | deny 규칙 추가 (도구 이름만 쓰면 도구 제거) | 한 번만 막고 싶을 때 |
| `--tools` | CLI | 사용할 수 있는 내장 도구 **자체**를 제한 (MCP 도구는 제외) | 읽기 전용 분석 세션 |
| PreToolUse hook | settings.json `hooks` | ✅ 스크립트로 명령 전체를 검사. 차단(exit 2)은 allow 규칙보다 우선 | 규칙으로 표현하기 어려운 검사 (URL 검증, 특정 파일 편집 차단) |
| Sandbox | `/sandbox`, `sandbox` 설정 | ✅✅ OS 수준 (셸 명령과 자식 프로세스의 파일·네트워크) | 간접 접근까지 차단. macOS·Linux·WSL2만 |
| Managed settings | 관리자 배포 | ✅✅ 사용자가 덮어쓸 수 없음 | 회사 보안 정책 |

**헷갈리기 쉬운 차이**

- `--allowedTools` 는 **승인을 생략**하는 것이고, `--tools` 는 도구를 **존재하게 할지**를 정한다. 도구를 아예 못 쓰게 하려면 `--tools` 나 `--disallowedTools` 를 쓴다.
- hook이 "allow"를 돌려줘도 deny·ask 규칙은 그대로 적용된다.
- 권한 규칙과 sandbox는 서로 보완한다: 규칙은 모든 도구를 다루고, sandbox는 셸 명령을 OS 수준에서 막는다. 프롬프트 인젝션으로 Claude의 판단이 흔들려도 sandbox 제한은 유지된다.

**예시 – CI에서 테스트만 허용**

```bash
claude -p "테스트를 실행하고 실패 원인을 요약해줘" --permission-mode dontAsk --allowedTools "Bash(mvn test *)" "Read"
```

```bash
# 읽기 전용 분석 세션
claude --tools "Read,Grep,Glob"
```

🔗 [CLI flags](https://code.claude.com/docs/en/cli-reference#cli-flags) · [Extend permissions with hooks](https://code.claude.com/docs/en/permissions#extend-permissions-with-hooks) · [How permissions interact with sandboxing](https://code.claude.com/docs/en/permissions#how-permissions-interact-with-sandboxing) · [Common setups](https://code.claude.com/docs/en/permission-modes#common-setups)

---

## 핵심 정리

1. **큰 작업은 Plan 모드로 탐색·계획한 뒤 구현한다.** diff를 한 문장으로 설명할 수 있는 작업이면 바로 시킨다.
2. **Claude에게 검증 수단(테스트·빌드)을 줘야** 스스로 반복해서 완성한다. 말이 아니라 증거를 받자.
3. **빗나가면 `Esc` / `/rewind` / `/clear` 로 빨리 되돌린다.** 체크포인트는 git을 대체하지 못하므로 자주 커밋한다.
4. **`-p` + 파이프 = Unix 도구.** `--output-format json` 의 `session_id` 로 세션을 이어가고, 스크립트·CI에는 `--bare` 를 쓴다.
5. **설정 우선순위는 Managed → CLI → local → project → user.** 배열은 합쳐진다. 실제 적용 상태는 `/status` 로 확인한다.
6. **권한은 deny → ask → allow 순서로 평가하고, 어디에 있든 deny가 이긴다.**
7. **민감 파일은 `.claudeignore` 가 아니라 `Read(...)` deny로 막는다.** 단 간접 읽기는 못 막으므로 sandbox(Windows는 WSL2), hook, 작업 디렉터리 밖 보관으로 보완한다.
8. **기본 시작 모드가 auto로 바뀌었다.** 회사 소스라면 `defaultMode` 를 명시하고 `bypassPermissions` 는 꺼둔다.
9. **`CLAUDE.md` 는 권고, permissions·hooks·sandbox·managed settings는 강제다.**
10. **Windows에서는 PowerShell 도구가 기본 셸이다.** 셸 규칙은 `Bash(...)` 와 `PowerShell(...)` 을 둘 다 적는다.

## 참고한 공식 문서

- [Best practices](https://code.claude.com/docs/en/best-practices)
- [Common workflows](https://code.claude.com/docs/en/common-workflows)
- [Run Claude Code programmatically](https://code.claude.com/docs/en/headless)
- [Manage sessions](https://code.claude.com/docs/en/sessions)
- [CLI reference](https://code.claude.com/docs/en/cli-reference)
- [Settings files and precedence](https://code.claude.com/docs/en/settings) · [All settings](https://code.claude.com/docs/en/settings-reference) · [Example settings files](https://code.claude.com/docs/en/settings-example)
- [Environment variables](https://code.claude.com/docs/en/env-vars)
- [Configure permissions](https://code.claude.com/docs/en/permissions) · [Choose a permission mode](https://code.claude.com/docs/en/permission-modes)
- [Sandboxing](https://code.claude.com/docs/en/sandboxing) · [Tools reference](https://code.claude.com/docs/en/tools-reference)
