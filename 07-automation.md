# 📔 7장. 자동화

> **기준**: 「클로드코드로 시작하는 실전 에이전틱 코딩」 CHAPTER 07 목차 + Claude Code 공식 문서([code.claude.com/docs](https://code.claude.com/docs), 2026-10-11 확인). 책과 달라진 부분은 **📌 현재 공식 문서 기준**, 공식 문서에 전용 설명이 없는 부분은 **🔎 공식 문서 원칙에서 정리** 로 표시했다.
>
> 관련 노트: [2장](./02-workflow-and-settings.md) (`-p` 기초, 파이프, 권한) · [3장](./03-agent-skills.md) (스킬) · [4장](./04-subagents.md) (서브에이전트) · [6장](./06-memory-and-sessions.md) (체크포인트, 세션)

---

## 7-1. 비대화형 모드와 스크립트 자동화

> `claude -p` 기본 사용법, 파이프, 출력 형식(`text`/`json`/`stream-json`), `-p` 에서 세션 이어가기는 [2장](./02-workflow-and-settings.md)에 정리했다. 여기서는 자동화에 필요한 플래그, 프로그램 연동, 주의점을 다룬다.

### 주요 플래그

| 플래그 | 용도 |
|---|---|
| `--bare` | hooks·skills·플러그인·MCP·auto memory·`CLAUDE.md` 자동 로드를 건너뜀. **어느 PC에서나 같은 결과**. `ANTHROPIC_API_KEY` 필요 |
| `--allowedTools "Read,Edit,Bash(git diff *)"` | 묻지 않고 허용할 도구 (권한 규칙 문법) |
| `--permission-mode` | `acceptEdits`(파일 수정 허용), `dontAsk`(물어야 할 것은 모두 거부, 잠긴 CI용), `auto`(분류기가 검토) |
| `--permission-prompts none` | 응답할 사람이 없을 때. 물어야 할 요청은 거부되고 Claude에게 재시도하지 말라고 알림 (v2.1.259+) |
| `--output-format json` / `stream-json` | 결과·세션 ID·비용을 JSON으로 |
| `--json-schema '…'` | 스키마에 맞춘 결과를 `structured_output` 으로 |
| `--append-system-prompt` | 기본 동작을 유지하며 지시 추가 (`--system-prompt` 는 통째로 교체) |
| `--max-turns`, `--max-budget-usd` | 턴 수·비용 상한 |
| `--mcp-config`, `--strict-mcp-config` | 지정한 MCP 서버만 사용 |
| `--settings <파일 또는 JSON>` | 이 실행에만 적용할 설정 (예: `'{"disableAllHooks": true}'`) |
| `--no-session-persistence` | 이 실행의 대화 기록을 저장하지 않음 |
| `--continue`, `--resume <id>` | 이전 대화 이어가기 |

> 📌 **현재 공식 문서 기준**: `--bare` 는 스크립트·SDK 호출에 **권장되는 모드이고, 앞으로 `-p` 의 기본값이 될 예정**이다. bare 모드에서는 구독 로그인(OAuth)을 쓰지 않으므로 API 키나 `apiKeyHelper` 가 필요하다.

**`--allowedTools` 의 공백 주의** — `Bash(git diff *)` 는 `git diff` 로 시작하는 명령을 허용한다. 공백 없이 `Bash(git diff*)` 로 쓰면 `git diff-index` 까지 허용된다.

🔗 [Run Claude Code programmatically](https://code.claude.com/docs/en/headless) · [Start faster with bare mode](https://code.claude.com/docs/en/headless#start-faster-with-bare-mode) · [CLI reference](https://code.claude.com/docs/en/cli-reference)

### 프로그래밍 활용

**① 종료 코드로 분기** — 성공 0, 실패는 0이 아님. 잘못된 플래그는 실행 전에 stderr로 보고한다. SIGTERM으로 멈추면 **143**.

**② JSON 결과 활용 (PowerShell)**

```powershell
$r = claude -p "pom.xml의 의존성 중 오래된 것을 찾아줘" --output-format json | ConvertFrom-Json
$r.result            # 응답 텍스트
$r.session_id        # 이어서 질문할 때 --resume 에 사용
$r.total_cost_usd    # 이번 실행 비용
```

**③ 스키마로 받기** — 후처리할 데이터는 스키마를 강제한다.

```bash
claude -p "src/main/java 아래 @Deprecated 메서드를 찾아줘" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"methods":{"type":"array","items":{"type":"string"}}},"required":["methods"]}' \
  | jq '.structured_output.methods'
```

**④ CI에서 설정이 제대로 로드됐는지 검사** — `stream-json` 의 첫 이벤트 `system/init` 에 `plugins`, `plugin_errors`, `mcp_servers`, `mcp_server_errors` 가 들어 있다. 잘못된 `--mcp-config` 항목은 **건너뛰고도 정상 종료**하므로 이 필드로 실패를 잡는다.

**⑤ `-p` 에서 쓸 수 있는 명령**

- 사용자가 호출하는 **스킬과 커스텀 명령은 동작**한다: 프롬프트에 `/스킬이름` 을 넣는다.
- `/login` 처럼 터미널 UI 전용 명령은 안 된다.
- `/model sonnet`, `/config key=value`, `/output-style 이름` 처럼 값을 인수로 주는 형태는 동작한다.

**⑥ 예시 – 스테이징된 변경으로 커밋 (허용 범위 최소화)**

```bash
claude -p "스테이징된 변경을 보고 적절한 메시지로 커밋해줘" \
  --allowedTools "Bash(git diff *),Bash(git log *),Bash(git status *),Bash(git commit *)"
```

🔗 [Get structured output](https://code.claude.com/docs/en/headless#get-structured-output) · [Fail CI when a plugin or MCP server doesn't load](https://code.claude.com/docs/en/headless#fail-ci-when-a-plugin-or-mcp-server-doesnt-load) · [Create a commit](https://code.claude.com/docs/en/headless#create-a-commit)

### 고려사항

> ⚠️ **가장 중요한 보안 주의 (공식 문서)**: `--bare` 없이 `-p` 를 실행하면 **워크스페이스 신뢰 대화상자도, MCP 서버별 승인도 없이** 프로젝트 `.claude/settings.json` 의 훅을 실행하고 `.mcp.json` 서버에 연결한다. 내가 작성하지 않은 저장소(외부 PR, 오픈소스)에 `claude -p` 를 돌리기 전에 `.claude/` 를 검토하거나, `--bare` 또는 `--settings '{"disableAllHooks": true}'` 를 쓴다.

| 항목 | 내용 |
|---|---|
| 파이프 입력 | 최대 10MB. 넘으면 파일로 저장하고 경로를 넘긴다 |
| 백그라운드 작업 대기 | 턴이 끝나도 백그라운드 작업을 기다린다. 기본 **10분** 유휴 대기 후 정리 |
| 비동기 훅 | `-p` 종료 시 실행 중인 async 훅은 **취소**된다 |
| 권한 프롬프트 | 사람이 없으므로 `--allowedTools`, `--permission-mode`, `--permission-prompts none` 으로 미리 정한다 |
| 재현성 | 팀원 PC의 `~/.claude` 설정이 끼어들지 않게 CI는 `--bare` |
| 비용 | `--max-turns`, `--max-budget-usd` 로 상한, JSON의 `total_cost_usd` 로 기록 |
| 플러그인 설치 지연 | CI에서 플러그인이 첫 턴에 없을 수 있음 → `CLAUDE_CODE_SYNC_PLUGIN_INSTALL=1` |

🔗 [Turn off permission prompts in unattended runs](https://code.claude.com/docs/en/headless#turn-off-permission-prompts-in-unattended-runs) · [Background tasks at exit](https://code.claude.com/docs/en/headless#background-tasks-at-exit) · [Workspace trust](https://code.claude.com/docs/en/hooks#workspace-trust)

---

## 7-2. 훅 시스템

### 훅이란

**① 개념** — Claude Code 수명 주기의 특정 시점에 **Claude Code가 직접 실행**하는 셸 명령, HTTP 요청, MCP 도구, 프롬프트, 에이전트.

**② 왜** — `CLAUDE.md`·출력 스타일은 "지시"라서 지켜지지 않을 수 있다. 훅은 Claude의 판단과 무관하게 **매번** 실행된다: 편집 후 포맷, 위험 명령 차단, 알림, 감사 로그.

**③ 구성 3단계**

1. **이벤트** 선택 (예: `PreToolUse`)
2. **매처 그룹**으로 언제 실행할지 필터 (예: `Bash` 도구만)
3. **핸들러**: 실행할 것 (명령, HTTP, MCP 도구, 프롬프트, 에이전트)

🔗 [Automate actions with hooks](https://code.claude.com/docs/en/hooks-guide) · [Hooks reference](https://code.claude.com/docs/en/hooks)

### 훅 이벤트

> 📌 **현재 공식 문서 기준**: 현재 훅 이벤트는 **30개 이상**이다. 자주 쓰는 PreToolUse, PostToolUse, UserPromptSubmit, Notification, Stop, SubagentStop, PreCompact, SessionStart, SessionEnd 외에 `PermissionRequest`, `PermissionDenied`, `PostToolUseFailure`, `PostToolBatch`, `InstructionsLoaded`, `ConfigChange`, `CwdChanged`, `FileChanged`, `WorktreeCreate`, `PostCompact`, `StopFailure`, `Elicitation` 등이 있다. 책의 이벤트 목록과 다르면 공식 목록을 기준으로 삼는다.

**실행 주기별 주요 이벤트**

| 주기 | 이벤트 | 시점 | 차단 가능 |
|---|---|---|---|
| 세션 | `SessionStart` | 세션 시작·재개 (`startup`, `resume`, `clear`, `compact`) | 아니오 (컨텍스트 추가) |
| 세션 | `SessionEnd` | 세션 종료 | 아니오 |
| 턴 | `UserPromptSubmit` | 프롬프트 제출 직후, Claude 처리 전 | 예 |
| 턴 | `Stop` | Claude가 응답을 마칠 때 | 예 (계속 작업시킴) |
| 턴 | `StopFailure` | API 에러로 턴 종료 | 아니오 |
| 도구 | `PreToolUse` | 도구 실행 전 | **예 (도구 호출 차단)** |
| 도구 | `PermissionRequest` | 권한 대화상자가 뜨기 직전 | JSON `decision` 으로 허용/거부 |
| 도구 | `PostToolUse` | 도구 성공 후 | 아니오 (이미 실행됨, stderr를 Claude에게 보여 줌) |
| 도구 | `PostToolUseFailure` | 도구 실패 후 | 아니오 |
| 기타 | `Notification` | 권한 대기, 입력 대기 알림 | 아니오 |
| 기타 | `SubagentStart` / `SubagentStop` | 서브에이전트 시작·종료 | Stop만 예 |
| 기타 | `PreCompact` / `PostCompact` | 압축 전·후 | Pre만 예 |
| 기타 | `InstructionsLoaded` | `CLAUDE.md`·룰 로드 시 | 아니오 |
| 기타 | `FileChanged` | 감시 중인 파일이 디스크에서 바뀔 때 | 아니오 |

🔗 [Hook lifecycle](https://code.claude.com/docs/en/hooks#hook-lifecycle) · [Hook events](https://code.claude.com/docs/en/hooks#hook-events)

### 종료 코드와 결정 제어

**① 종료 코드 (명령 훅)**

| 종료 코드 | 의미 | 결과 |
|---|---|---|
| **0** | 성공 | stdout의 JSON이 있으면 적용. 대부분 이벤트에서 stdout은 debug 로그에만 기록 (`UserPromptSubmit`, `SessionStart` 등은 Claude 컨텍스트에 추가) |
| **2** | 차단 오류 | 차단 가능한 이벤트면 동작을 멈춤. **stderr가 차단 사유**로 전달됨 |
| 그 밖 | 비차단 오류 | 동작은 계속. 트랜스크립트에 `<hook name> hook error` 표시 |

> ⚠️ **exit 1은 차단이 아니다.** 막으려면 반드시 2. 또 exit 0의 stderr는 Claude가 보지 못한다.

**② JSON 출력 (exit 0 + stdout에 JSON 하나만)** — 더 세밀한 제어.

| 필드 | 용도 |
|---|---|
| `continue: false` 와 `stopReason` | Claude를 완전히 멈춤 |
| `systemMessage` | 사용자에게 경고 표시 |
| `decision: "block"` 과 `reason` | UserPromptSubmit, PostToolUse, Stop 등에서 차단·피드백 |
| `hookSpecificOutput.permissionDecision` | PreToolUse: `allow` / `deny` / `ask` / `defer` |
| `hookSpecificOutput.updatedInput` | PreToolUse: 실행 전 도구 인수 수정 |
| `hookSpecificOutput.additionalContext` | Claude 컨텍스트에 문자열 추가 (최대 10,000자) |

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "운영 설정 파일은 수정할 수 없습니다"
  }
}
```

**③ 주의**

- 한 훅에서는 **종료 코드 방식이나 JSON 방식 중 하나**만 쓴다.
- 셸 프로필(`.bashrc` 등)이 무언가 `echo` 하면 stdout이 `{` 로 시작하지 않아 JSON이 무시된다.
- `PreToolUse` 의 `deny` 는 `bypassPermissions` 모드에서도 막는다. 반대로 `allow` 는 설정의 deny 규칙을 넘지 못한다.

🔗 [Exit code output](https://code.claude.com/docs/en/hooks#exit-code-output) · [JSON output](https://code.claude.com/docs/en/hooks#json-output) · [Decision control](https://code.claude.com/docs/en/hooks#decision-control)

### 구성 파일과 우선순위

| 위치 | 범위 | 공유 |
|---|---|---|
| `~/.claude/settings.json` | 내 모든 프로젝트 | 아니오 |
| `.claude/settings.json` | 이 프로젝트 | 예 (커밋) |
| `.claude/settings.local.json` | 이 프로젝트, 나만 | 아니오 |
| 관리형 정책 설정 | 조직 전체 | 관리자 |
| 플러그인 `hooks/hooks.json` | 플러그인이 켜져 있을 때 | 예 |
| 스킬 frontmatter | 스킬 호출 후 세션 끝까지 | 예 |
| 서브에이전트 frontmatter | 그 서브에이전트 실행 중에만 | 예 |

**규칙**

- 훅은 다른 설정과 달리 **덮어쓰지 않고 합쳐진다.** 사용자·프로젝트·로컬 훅이 모두 실행된다. 여러 파일에 같은 핸들러가 있으면 한 번만 실행된다.
- 일치하는 훅은 **모두 병렬**로 실행된다.
- `"disableAllHooks": true` 로 잠시 끌 수 있다. 단 관리형 훅은 관리형 설정에서만 끌 수 있다.
- 관리자가 `allowManagedHooksOnly` 를 켜면 사용자·프로젝트·플러그인 훅이 막힌다.
- **대화형 세션은 워크스페이스 신뢰 전에 훅을 실행하지 않는다** (내 `~/.claude/settings.json` 훅도 포함).
- `/hooks` 메뉴로 설정된 훅과 출처를 읽기 전용으로 확인한다. 파일을 직접 고치면 보통 자동 반영된다.

🔗 [Hook locations](https://code.claude.com/docs/en/hooks#hook-locations) · [Disable or remove hooks](https://code.claude.com/docs/en/hooks#disable-or-remove-hooks)

### 매처

**① 매처 해석 규칙**

| 매처 값 | 해석 | 예 |
|---|---|---|
| `"*"`, `""`, 생략 | 전부 | 모든 경우 |
| 글자·숫자·`_`·`-`·공백·`,`·`\|` 만 | 정확한 문자열 (목록) | `Bash`, `Edit\|Write`, `Edit, Write` |
| 그 외 문자 포함 | JavaScript 정규식 (부분 일치) | `^Notebook`, `mcp__memory__.*` |

- 대소문자를 구분한다.
- `Edit.*` 는 `NotebookEdit` 에도 맞는다. 전체 일치는 `^Edit$`.
- 이벤트마다 매처가 보는 필드가 다르다: 도구 이벤트는 도구 이름, `SessionStart` 는 `startup`/`resume`/`clear`/`compact`, `Notification` 은 알림 종류, `PreCompact` 는 `manual`/`auto`. `UserPromptSubmit`, `Stop` 은 매처를 지원하지 않는다.

**② `if` 필드로 더 좁히기** — 핸들러별로 권한 규칙 문법을 쓴다.

```json
{ "type": "command", "if": "Bash(git push *)", "command": "~/.claude/hooks/check-branch.sh" }
```

`if` 는 규칙 하나만 쓸 수 있고, 판별이 애매하면 **실행하는 쪽**으로 동작한다 (최선 노력). 반드시 막아야 하면 권한 규칙을 쓴다.

**③ MCP 도구 매칭** — 이름 규칙은 `mcp__<서버>__<도구>`. 서버 전체는 `mcp__github__.*` (**`.*` 필수**: `mcp__github` 만 쓰면 정확한 문자열로 비교해 아무것도 안 맞는다). 플러그인 MCP 서버는 `mcp__plugin_<플러그인>_<서버>__<도구>`.

🔗 [Matcher patterns](https://code.claude.com/docs/en/hooks#matcher-patterns) · [Match MCP tools](https://code.claude.com/docs/en/hooks#match-mcp-tools)

### 프롬프트 기반 훅과 에이전트 기반 훅

| | 프롬프트 훅 (`type: "prompt"`) | 에이전트 훅 (`type: "agent"`) |
|---|---|---|
| 동작 | 훅 입력과 프롬프트를 모델에 보내 **한 번에** 판단 | 서브에이전트가 Read·Grep·Glob 등으로 **조사 후** 판단 (최대 50턴) |
| 응답 | `{"ok": true}` 또는 `{"ok": false, "reason": "…"}` | 같음 |
| 기본 타임아웃 | 30초 | 60초 |
| 상태 | 정식 | **실험적** (운영에는 명령 훅 권장) |
| 쓰는 곳 | "모든 작업이 끝났나?" 같은 판단 | "테스트가 실제로 통과하나?"처럼 파일·출력 확인이 필요한 판단 |

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",
            "prompt": "Verify that all unit tests pass. Run the test suite and check the results. $ARGUMENTS",
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

- `$ARGUMENTS` 자리에 훅 입력 JSON이 들어간다 (없으면 끝에 붙음).
- `Stop` 에서 `ok: false` 면 사유가 Claude의 다음 지시가 되어 작업을 계속한다.
- 지원 이벤트가 정해져 있다: `PreToolUse`, `PostToolUse`, `Stop`, `UserPromptSubmit` 등은 지원, `SessionStart`·`Notification` 등은 미지원.

> ⚠️ Stop 훅이 계속 막으면 무한 반복이 될 수 있다. Claude Code는 **연속 8회** 차단되면 Stop 훅을 무시한다. 명령 훅은 입력의 `stop_hook_active` 를 확인해 이미 한 번 계속시켰으면 `exit 0` 한다.

🔗 [Prompt-based hooks](https://code.claude.com/docs/en/hooks#prompt-based-hooks) · [Agent-based hooks](https://code.claude.com/docs/en/hooks#agent-based-hooks) · [Stop hook hits the block cap](https://code.claude.com/docs/en/hooks-guide#stop-hook-hits-the-block-cap)

### 비동기 훅

**① 개념** — `"async": true` 인 **명령 훅**은 백그라운드에서 실행되고 Claude는 기다리지 않는다. 배포, 테스트 스위트, 외부 API 호출처럼 오래 걸리는 작업용.

**② 동작**

- 차단·제어는 할 수 없다.
- 끝나면 JSON의 `additionalContext`, `systemMessage` 가 **다음 턴**에 Claude에게 전달된다 (사용자에게는 표시 안 됨).
- `timeout` 을 적용하지 않는다.
- `asyncRewake: true` 는 백그라운드로 돌다가 **exit 2로 끝나면 Claude를 즉시 깨운다** (세션이 유휴 상태여도).
- `-p` 가 끝나면 실행 중인 async 훅은 취소된다.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/run-tests-async.sh", "async": true }
        ]
      }
    ]
  }
}
```

🔗 [Run hooks in the background](https://code.claude.com/docs/en/hooks#run-hooks-in-the-background)

### 프론트매터 훅과 에이전트 범위 훅

스킬·서브에이전트 파일의 YAML frontmatter에 settings와 같은 형식으로 훅을 넣을 수 있다.

```yaml
---
name: db-migration
description: Flyway 마이그레이션 스크립트를 작성하고 검증한다
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/block-prod-db.sh"
---
```

| | 스킬 frontmatter 훅 | 서브에이전트 frontmatter 훅 |
|---|---|---|
| 등록 시점 | 스킬을 호출할 때 | 서브에이전트가 실행될 때 |
| 유지 기간 | **그 뒤 세션 끝까지** (`once: true` 면 첫 성공 후 제거) | **그 서브에이전트가 끝나면 제거** |
| 특이점 | — | `Stop` 훅은 `SubagentStop` 으로 바뀐다 |
| 신뢰 | 프로젝트 스킬은 settings 훅과 같은 규칙 | 워크스페이스 신뢰 후에만 실행 (`-p` 는 신뢰로 치지 않음) |

> 📌 **현재 공식 문서 기준**: **플러그인에 들어 있는 에이전트**는 frontmatter의 `hooks`, `mcpServers`, `permissionMode` 를 무시한다. 플러그인에서는 `hooks/hooks.json` 으로 넣는다.

🔗 [Hooks in skills and agents](https://code.claude.com/docs/en/hooks#hooks-in-skills-and-agents)

### 실용 예제

**① 입력이 필요할 때 Windows 알림** (`~/.claude/settings.json`, 공식 예시)

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "powershell.exe -Command \"[System.Reflection.Assembly]::LoadWithPartialName('System.Windows.Forms'); [System.Windows.Forms.MessageBox]::Show('Claude Code needs your attention', 'Claude Code')\""
          }
        ]
      }
    ]
  }
}
```

매처를 `permission_prompt`(권한 대기 약 6초) 또는 `idle_prompt`(응답 후 약 60초 입력 없음)로 좁힐 수 있다.

**② 보호 파일 수정 차단 — PowerShell 버전**

> 🔎 **공식 문서 원칙에서 정리**: 공식 예제는 Bash + `jq` 다. 아래는 같은 로직을 Windows PowerShell로 옮긴 것이다 (`shell: "powershell"` 과 `$env:CLAUDE_PROJECT_DIR` 사용법은 공식 문서 기준).

```powershell
# .claude/hooks/protect-files.ps1
$in = [Console]::In.ReadToEnd() | ConvertFrom-Json
$path = ($in.tool_input.file_path -replace '\\', '/')
$protected = @('.env', 'application-prod', '.git/', 'WEB-INF/web.xml')
foreach ($p in $protected) {
    if ($path -like "*$p*") {
        [Console]::Error.WriteLine("Blocked: $path matches protected pattern '$p'")
        exit 2
    }
}
exit 0
```

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "shell": "powershell",
            "command": "& \"$env:CLAUDE_PROJECT_DIR\\.claude\\hooks\\protect-files.ps1\""
          }
        ]
      }
    ]
  }
}
```

> ⚠️ PowerShell 훅에서 `$CLAUDE_PROJECT_DIR`(맨 이름)를 쓰면 `$null` 로 해석된다. `${CLAUDE_PROJECT_DIR}` 또는 `$env:CLAUDE_PROJECT_DIR` 로 쓴다.

**③ 편집 후 자동 포맷** (공식 예시는 Prettier)

```json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "Edit|Write", "hooks": [ { "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write" } ] }
    ]
  }
}
```

Bash 명령이 파일을 바꿔도 포맷하려면 `FileChanged` 훅을 쓴다.

**④ 압축 후 핵심 맥락 다시 넣기** — `SessionStart` 이벤트, 매처 `compact`. stdout이 Claude 컨텍스트에 추가된다.

```json
{
  "hooks": {
    "SessionStart": [
      { "matcher": "compact", "hooks": [ { "type": "command", "command": "echo '알림: 빌드는 mvn -q, 커밋 전 mvn test. 현재 작업: 환불 기능'" } ] }
    ]
  }
}
```

🔗 [What you can automate](https://code.claude.com/docs/en/hooks-guide#what-you-can-automate)

### 실행 특성과 디버깅

**① 실행 특성**

| 항목 | 내용 |
|---|---|
| 기본 타임아웃 | `command`·`http`·`mcp_tool` **10분**, `prompt` 30초, `agent` 60초 (`UserPromptSubmit` 은 30초로 낮춤) |
| `SessionEnd` | 모든 훅이 1.5초 예산 공유 (설정한 timeout만큼, 최대 60초까지 늘어남) |
| 실행 셸 | 기본 bash. Windows는 Git Bash, 없으면 PowerShell. `shell` 필드로 지정 |
| exec 형식 | `args` 를 쓰면 셸 없이 직접 실행 (따옴표 문제 없음). Windows에서는 `.exe` 여야 함 (`.cmd`·`.bat` 불가) |
| 여러 훅이 `updatedInput` 반환 | 마지막에 끝난 것이 적용 (병렬이라 순서 비결정적) → 하나만 수정하게 |
| 출력 상한 | `additionalContext`, `systemMessage`, stdout 각각 10,000자 |

**② 디버깅 순서**

1. `/hooks` 에 그 이벤트 아래 훅이 보이는지
2. 매처 철자·대소문자
3. 모의 입력으로 직접 실행: `echo '{"tool_name":"Bash","tool_input":{"command":"ls"}}' | ./my-hook.sh; echo $?`
4. `claude --debug` → `~/.claude/debug/<session-id>.txt` 에 매칭·종료 코드·출력 기록 (터미널에는 안 나옴). 더 자세히는 `CLAUDE_CODE_DEBUG_LOG_LEVEL=verbose`
5. `command not found` → 절대 경로나 `${CLAUDE_PROJECT_DIR}`, `jq` 설치 확인

🔗 [Limitations and troubleshooting](https://code.claude.com/docs/en/hooks-guide#limitations-and-troubleshooting) · [Debug hooks](https://code.claude.com/docs/en/hooks#debug-hooks)

### 체크포인팅 보완 훅

> 🔎 **공식 문서 원칙에서 정리**: 공식 문서에 "체크포인팅 보완 훅" 예제는 없다. 체크포인트가 **Bash 변경, 대부분의 서브에이전트 편집, 외부 편집을 추적하지 않는다**는 공식 한계를 훅으로 보완하는 아이디어다. 아래 git 명령은 git 자체 기능이다.

**아이디어 1 — 위험한 Bash 명령 전에 작업 트리 스냅샷 (제안)**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(rm *)",
            "command": "ref=$(git stash create) && [ -n \"$ref\" ] && git stash store -m \"claude-snapshot $(date +%H%M%S)\" \"$ref\"; exit 0"
          }
        ]
      }
    ]
  }
}
```

- `git stash create` 는 작업 트리를 **바꾸지 않고** 현재 변경을 커밋 객체로 만든다. 복구는 `git stash list` → `git stash apply stash@{N}`.
- 추적되지 않은(untracked) 새 파일은 포함되지 않는다는 git의 한계가 있다.
- 실패해도 작업을 막지 않도록 `exit 0` 으로 끝낸다.

**아이디어 2 — 턴이 끝날 때 상태 기록** — `Stop` 훅에서 `git status --short` 를 로그 파일에 남기면, 체크포인트가 놓친 변경(빌드 산출물 등)을 나중에 확인할 수 있다.

> ⚠️ 자동 커밋 훅은 공유 브랜치를 어지럽힐 수 있다. 커밋은 사람이 확인한 뒤에 하는 것을 기본으로 둔다.

🔗 [Checkpointing: Limitations](https://code.claude.com/docs/en/checkpointing#limitations) · [Hook handler fields](https://code.claude.com/docs/en/hooks#hook-handler-fields)

### HTTP 훅

**① 개념** — 훅 입력 JSON을 URL로 **POST** 한다. 응답 본문은 명령 훅과 같은 JSON 출력 형식이다. 사내 정책 서버, 감사 로그 수집기에 연결할 때 쓴다.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "http",
            "url": "http://localhost:8080/hooks/pre-tool-use",
            "timeout": 30,
            "headers": { "Authorization": "Bearer $MY_TOKEN" },
            "allowedEnvVars": ["MY_TOKEN"]
          }
        ]
      }
    ]
  }
}
```

**② 응답 처리**

| 응답 | 결과 |
|---|---|
| 2xx + 빈 본문 | 성공 (exit 0과 같음) |
| 2xx + JSON | JSON 출력으로 해석 |
| 2xx + 일반 텍스트 | 비차단 오류 |
| 2xx 아님, 연결 실패 | **비차단 오류 — 계속 진행** |

> ⚠️ **상태 코드만으로는 차단할 수 없다.** 막으려면 2xx로 결정 필드가 담긴 JSON을 돌려준다. 서버가 죽어도 동작이 계속되므로, 실패 시 막아야 하면 `onFailure: "block"`(v2.1.295+)을 쓴다.

**③ 보안 설정**

- 헤더의 환경 변수는 `allowedEnvVars` 에 적은 것만 치환된다.
- 관리자는 `allowedHttpHookUrls`(허용 URL 목록), `httpHookAllowedEnvVars` 로 모든 출처의 HTTP 훅을 제한할 수 있다.

🔗 [HTTP hook fields](https://code.claude.com/docs/en/hooks#http-hook-fields) · [HTTP response handling](https://code.claude.com/docs/en/hooks#http-response-handling)

---

## 7-3. MCP (Model Context Protocol)

**개념** — MCP는 AI 도구를 외부 시스템(이슈 트래커, DB, 디자인 도구 등)에 연결하는 개방형 표준이다. MCP 서버를 연결하면 Claude가 그 도구를 직접 호출한다.

### 전송 방식

| 방식 | 명령 | 언제 |
|---|---|---|
| **HTTP** (권장) | `claude mcp add --transport http notion https://mcp.notion.com/mcp` | 원격 클라우드 서비스 |
| SSE | `claude mcp add --transport sse asana https://mcp.asana.com/sse` | **deprecated**. SSE만 제공하는 서비스 |
| **stdio** | `claude mcp add --transport stdio db -- npx -y @bytebase/dbhub --dsn "…"` | 내 PC에서 프로세스로 실행 |
| WebSocket | `.mcp.json` 또는 `claude mcp add-json` 에 `"type": "ws"` | 원격 WebSocket 서버 |

**stdio의 `--`** — `--` 앞은 Claude Code 옵션(`--transport`, `--env`, `--scope`), 뒤는 서버 실행 명령 그대로다. `--` 가 없으면 서버의 `--port` 같은 플래그를 Claude Code가 자기 옵션으로 해석한다.

> 📌 **현재 공식 문서 기준**: SSE 전송은 **deprecated** 다. SSE만 제공하는 서비스도 `--transport http` 로 추가하면 HTTP를 먼저 시도하고 필요하면 SSE로 전환한다. 현재 MCP 문서에는 Windows에서 `npx` 를 `cmd /c` 로 감싸라는 안내가 없다. 책에 그런 안내가 있다면 지금 환경에서 직접 확인하자.

🔗 [Installing MCP servers](https://code.claude.com/docs/en/mcp#installing-mcp-servers)

### 주요 서버 사례

공식 문서의 실전 예제 (서비스 목록은 자주 바뀌므로 공식 문서의 서버 찾기 절을 참고):

| 용도 | 예 |
|---|---|
| 코드 리뷰, 이슈 | GitHub 원격 서버: `claude mcp add --transport http github https://api.githubcopilot.com/mcp/ --header "Authorization: Bearer YOUR_GITHUB_PAT"` |
| DB 조회 | DBHub(`@bytebase/dbhub`) stdio 서버 + `--dsn` (공식 권장: **읽기 전용 DB 계정**) |
| 문서·업무 도구 | Notion, Asana 등 원격 HTTP 서버 (OAuth 인증) |

**사용 예** — "PR #456 리뷰하고 개선점 제안해줘", "주문 테이블 스키마 보여줘", "90일 동안 구매가 없는 고객 찾아줘"

🔗 [Practical examples](https://code.claude.com/docs/en/mcp#practical-examples) · [Find and build MCP servers](https://code.claude.com/docs/en/mcp#find-and-build-mcp-servers)

### 설정과 구현

**① 설치 범위 (scope)**

| 범위 | 로드되는 곳 | 팀 공유 | 저장 위치 |
|---|---|---|---|
| **local** (기본) | 현재 프로젝트만 | 아니오 | `~/.claude.json` (프로젝트별 항목) |
| **project** | 현재 프로젝트만 | **예 (git)** | 프로젝트 루트 `.mcp.json` |
| **user** | 내 모든 프로젝트 | 아니오 | `~/.claude.json` |

> ⚠️ MCP의 "local 범위"는 일반 설정의 `.claude/settings.local.json` 과 다르다. MCP local은 **홈 디렉터리의 `~/.claude.json`** 에 저장된다.

**② 같은 서버가 여러 곳에 있으면** — local → project → user → 플러그인 → claude.ai 커넥터 순으로 **우선순위가 높은 정의 하나만** 쓴다 (필드를 섞지 않음). 조직의 `managedMcpServers` 는 그보다 위.

**③ 관리 명령**

```bash
claude mcp list                 # 목록 + 상태 (✔ Connected, ! Needs authentication, ✘ Failed)
claude mcp get github           # 상세
claude mcp remove github        # 제거 (저장된 OAuth 토큰도 삭제)
claude mcp reset-project-choices  # .mcp.json 승인 선택 초기화
/mcp                            # 세션 안에서 상태 확인, OAuth 인증, 켜기/끄기
```

**④ 팀 공유용 `.mcp.json` — 비밀은 환경 변수로**

```json
{
  "mcpServers": {
    "api-server": {
      "type": "http",
      "url": "${API_BASE_URL:-https://api.example.com}/mcp",
      "headers": { "Authorization": "Bearer ${API_KEY}" }
    }
  }
}
```

- `${VAR}`, `${VAR:-기본값}` 을 `command`, `args`, `env`, `url`, `headers` 에서 쓸 수 있다.
- 원격 서버의 `url`·`headers` 에서 `ANTHROPIC_API_KEY`, `HTTPS_PROXY`, `NPM_TOKEN` 같은 **자격 증명 변수는 빈 값으로 읽힌다** (프로젝트 파일이 내 키를 외부로 보내지 못하게).
- 프로젝트 `.mcp.json` 서버는 대화형 세션에서 **처음 사용 전 승인**을 받는다.

**⑤ 인증** — 원격 서버가 401/403을 주면 `/mcp` 에서 OAuth 로그인을 진행한다. 고정 헤더가 필요하면 `--header`, 동적 토큰이면 `headersHelper`.

🔗 [MCP installation scopes](https://code.claude.com/docs/en/mcp#mcp-installation-scopes) · [Environment variable expansion](https://code.claude.com/docs/en/mcp#environment-variable-expansion-in-mcp-json) · [Authenticate with remote MCP servers](https://code.claude.com/docs/en/mcp#authenticate-with-remote-mcp-servers)

### 훅과 MCP 연동

**① MCP 도구에 훅 걸기** — MCP 도구도 일반 도구처럼 `PreToolUse`·`PostToolUse` 등에서 매칭된다.

```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "mcp__db__.*", "hooks": [ { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/audit-db-query.sh" } ] },
      { "matcher": "mcp__.*__write.*", "hooks": [ { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/validate-mcp-write.sh" } ] }
    ]
  }
}
```

**② 훅에서 MCP 도구 호출하기 (`type: "mcp_tool"`)** — 훅 핸들러가 설정된 MCP 서버의 도구를 직접 호출한다. 문자열 값에 `${tool_input.file_path}` 처럼 훅 입력을 넣을 수 있다.

```json
{ "type": "mcp_tool", "server": "my_server", "tool": "security_scan", "input": { "path": "${tool_input.file_path}" } }
```

**③ MCP 사용자 입력 요청** — 서버가 도구 실행 중 사용자 입력을 요청하면 `Elicitation` / `ElicitationResult` 이벤트가 발생한다. 비대화형 실행에서 자동 응답할 때 쓴다.

🔗 [Match MCP tools](https://code.claude.com/docs/en/hooks#match-mcp-tools) · [MCP tool hook fields](https://code.claude.com/docs/en/hooks#mcp-tool-hook-fields)

### 유의 사항

| 항목 | 내용 |
|---|---|
| **신뢰** | 연결 전에 서버를 신뢰할 수 있는지 확인. 외부 콘텐츠를 가져오는 서버는 **프롬프트 인젝션** 위험이 있다 |
| **`-p` 와 프로젝트 서버** | `-p`·SDK·클라우드 세션은 승인 없이 `.mcp.json` 서버를 로드한다 → `disabledMcpjsonServers`, `--strict-mcp-config` 로 통제 |
| **출력 크기** | 도구 출력 1만 토큰 초과 시 경고, 기본 최대 **25,000 토큰** (`MAX_MCP_OUTPUT_TOKENS`). 넘으면 파일로 저장하고 경로만 전달 |
| **컨텍스트** | 도구 검색(tool search)이 기본: 시작 시 도구 이름만 로드하고 정의는 필요할 때 → 서버가 많아도 컨텍스트 부담이 작다 |
| **DB 서버** | 읽기 전용 계정 사용 (공식 예제 권고) |
| **비밀 값** | `.mcp.json` 에 토큰을 직접 쓰지 말고 `${VAR}` 로. 붙여 넣은 토큰 끝의 숨은 공백은 경고로 알려 준다 |
| **끄기** | 제거하지 않고 `/mcp` 에서 비활성화 가능 (프로젝트별로 `~/.claude.json` 에 기록) |

🔗 [MCP output limits and warnings](https://code.claude.com/docs/en/mcp#mcp-output-limits-and-warnings) · [Scale with MCP tool search](https://code.claude.com/docs/en/mcp#scale-with-mcp-tool-search) · [Security](https://code.claude.com/docs/en/security)

---

## 7-4. 플러그인

### 플러그인 구조

**① 개념** — 스킬, 에이전트, 훅, MCP 서버 등을 **한 단위로 묶어** 설치·배포하는 디렉터리. 대부분 마켓플레이스(플러그인 목록 JSON)에서 설치한다.

**② 언제 플러그인을 쓰나** — 스킬·훅·MCP는 플러그인 없이도 동작한다. **여러 개를 묶어 팀에 배포하거나, 여러 프로젝트에 설치하거나, 업데이트를 받고 싶을 때** 플러그인으로 만든다.

**③ 디렉터리 구조**

```text
my-plugin/
├── .claude-plugin/
│   └── plugin.json        # 매니페스트 (이 폴더에는 plugin.json 만)
├── skills/review/SKILL.md # 스킬
├── commands/              # (예전 형식, 새로 만들 때는 skills/)
├── agents/                # 서브에이전트
├── hooks/hooks.json       # 훅
├── .mcp.json              # MCP 서버
├── .lsp.json              # LSP 서버
├── output-styles/         # 출력 스타일
├── bin/                   # Bash 도구 PATH에 추가되는 실행 파일
└── settings.json          # 기본 설정
```

```json
{
  "name": "team-quality",
  "displayName": "Team Quality",
  "description": "코드 리뷰 스킬, 보안 리뷰 에이전트, 편집 후 포맷 훅",
  "version": "1.0.0",
  "author": { "name": "Platform Team" }
}
```

> ⚠️ 컴포넌트를 `.claude-plugin/` 안에 두면 로드되지 않는다. 플러그인 루트의 `CLAUDE.md` 도 로드되지 않으므로 지시는 스킬로 넣는다.

🔗 [Plugins overview](https://code.claude.com/docs/en/plugins/overview) · [Plugin layout](https://code.claude.com/docs/en/plugins/create#plugin-layout)

### 컴포넌트 유형

| 컴포넌트 | 위치 | 로드 후 이름 |
|---|---|---|
| 스킬 | `skills/<이름>/SKILL.md` | `/플러그인:이름` |
| 명령 (예전 형식) | `commands/<파일>.md` | `/플러그인:파일` |
| 에이전트 | `agents/<이름>.md` | `플러그인:이름` (`@agent-플러그인:이름` 으로 호출) |
| 훅 | `hooks/hooks.json` | 플러그인이 켜지면 바로 등록 (스킬 사용 여부와 무관) |
| MCP 서버 | `.mcp.json` | `/mcp` 에 `plugin:플러그인:서버`, 도구는 `mcp__plugin_플러그인_서버__도구` |
| LSP 서버 | `.lsp.json` | 진단·코드 탐색 |
| 출력 스타일 | `output-styles/` | 스타일 목록에 추가 (`force-for-plugin` 가능) |
| 실행 파일 | `bin/` | Bash 도구에서 이름만으로 실행 |
| 기본 설정 | `settings.json` | 플러그인 기본값 |

**경로 변수** — `${CLAUDE_PLUGIN_ROOT}`(설치된 플러그인 디렉터리, **버전마다 바뀜**), `${CLAUDE_PLUGIN_DATA}`(업데이트에도 유지되는 데이터 디렉터리). 오래 보관할 파일은 DATA에 둔다.

> 📌 **현재 공식 문서 기준**: 현재 플러그인 컴포넌트는 스킬·명령·에이전트·훅·MCP 외에 **LSP 서버, 출력 스타일, 실행 파일(`bin/`), 기본 설정, 모니터, 채널, 훅 모듈(mod)** 까지 포함한다. 명령(`commands/`)은 예전 형식이고 새로 만들 때는 스킬을 쓴다.

🔗 [Add components to a plugin](https://code.claude.com/docs/en/plugins/components)

### 프로젝트 설정과의 관계

> 🔎 **공식 문서 원칙에서 정리**: 공식 문서의 "언제 플러그인이 필요한가"와 컴포넌트 규칙을 `.claude/` 직접 설정과 비교했다.

| | `.claude/` 에 직접 | 플러그인 |
|---|---|---|
| 스킬 이름 | `/review` | `/team-quality:review` (네임스페이스) |
| 범위 | 그 저장소 | 설치한 범위(사용자·프로젝트·로컬) |
| 배포·업데이트 | git pull | 마켓플레이스에서 버전 업데이트 |
| 여러 저장소 공유 | 복사 필요 | 한 번 만들고 여러 곳에 설치 |
| 적합한 것 | 그 저장소에만 해당하는 규칙·스킬 | 팀·조직 공통 도구 묶음 |

**함께 쓸 때** — 훅은 settings 훅과 **합쳐져** 모두 실행된다. MCP 서버는 같은 서버가 있으면 사용자 설정(local/project/user)이 플러그인보다 우선한다.

> 💡 켜진 플러그인은 **모든 세션에 비용이 든다**: Claude가 스스로 호출할 수 있는 스킬·에이전트의 이름과 설명이 매 턴 컨텍스트에 들어가고, MCP 서버가 함께 실행된다. 안 쓰는 플러그인은 끈다.

🔗 [Decide whether you need a plugin](https://code.claude.com/docs/en/plugins/overview#decide-whether-you-need-a-plugin) · [What an enabled plugin adds to your sessions](https://code.claude.com/docs/en/plugins/overview#what-an-enabled-plugin-adds-to-your-sessions)

### 설치 범위

| 범위 | 대상 | 기록 위치 |
|---|---|---|
| **user** | 이 PC의 모든 프로젝트 | `~/.claude/settings.json` 의 `enabledPlugins` |
| **project** | 이 저장소의 모든 사람 | `.claude/settings.json` (커밋) |
| **local** | 이 저장소, 나만 | `.claude/settings.local.json` |

- 같은 플러그인이 여러 범위에 있으면 **local > project > user**. 관리형 설정은 모두보다 우선 (`true` 강제, `false` 차단).
- 프로젝트가 켠 플러그인을 나만 끄려면 `.claude/settings.local.json` 에 `false`.
- 터미널, 데스크톱 앱, VS Code 확장은 같은 설정 파일을 읽으므로 user 범위 설치는 셋 모두에 적용된다.

> ⚠️ **project 범위로 켜도 팀원 PC에 자동 다운로드되지는 않는다.** 외부 소스 플러그인은 각자 설치해야 하며, 설치 전에는 `/plugin` **Errors** 탭에 `is enabled in project settings but isn't installed here` 가 뜬다 (예외는 아래 "팀 워크플로" 참고).

🔗 [Choose an install scope](https://code.claude.com/docs/en/plugins/install#choose-an-install-scope) · [Find where a plugin is enabled](https://code.claude.com/docs/en/plugins/loading#find-where-a-plugin-is-enabled)

### LSP 서버 지원

**① 무엇을 얻나**

- **편집 후 진단**: Claude가 파일을 고칠 때마다 타입 에러, import 누락, 문법 오류를 컴파일 없이 받는다 (`Found N new diagnostic issues`, `Ctrl+O` 로 상세).
- **코드 탐색**: 텍스트 검색 대신 `LSP` 도구로 정의·참조를 심볼 단위로 찾는다 (읽기 전용).

**② Java 프로젝트 설정 (공식 코드 인텔리전스 플러그인)**

1. 언어 서버 `jdtls` 를 설치하고 `claude` 를 실행하는 셸의 PATH에 둔다 (PowerShell 확인: `Get-Command jdtls`). 설치 방법은 플러그인 README 참고.
2. `/plugin install jdtls-lsp@claude-plugins-official`
3. Claude에게 Java 파일에 일부러 타입 에러를 넣고 고치게 해서 진단 줄이 나오는지 확인한다.

공식 플러그인이 있는 언어: C/C++, C#, Go, **Java(`jdtls-lsp`)**, Kotlin, Lua, PHP, Python, Ruby, Rust, Swift, TypeScript/JavaScript 등.

**③ 직접 만들기 (`.lsp.json`)**

```json
{
  "gopls": {
    "command": "gopls",
    "args": ["serve"],
    "extensionToLanguage": { ".go": "go" }
  }
}
```

- 플러그인은 **연결만 설정**하고 언어 서버 바이너리는 설치하지 않는다. 바이너리가 없으면 Errors 탭에 `Executable not found in $PATH`.
- 바이너리가 이미 PATH에 있으면 Claude Code가 **LSP plugin recommendation** 대화상자로 설치를 제안한다.
- 클라우드 세션에서는 언어 서버를 시작하지 않는다.

🔗 [Code intelligence plugins](https://code.claude.com/docs/en/plugins/code-intelligence) · [LSP servers](https://code.claude.com/docs/en/plugins/components#lsp-servers)

### 앤트로픽 공식 마켓플레이스

| | Official | Community | Demo |
|---|---|---|---|
| 저장소 | `anthropics/claude-plugins-official` | `anthropics/claude-plugins-community` | `anthropics/claude-code` |
| 설치 시 이름 | `@claude-plugins-official` | `@claude-community` | `@claude-code-plugins` |
| 내용 | Anthropic 유지 + 파트너 플러그인 | 작성자가 제출한 서드파티 | 예제 플러그인 |
| 추가 방법 | **첫 대화형 세션에서 자동 추가** | `/plugin marketplace add anthropics/claude-plugins-community` | `/plugin marketplace add anthropics/claude-code` |
| 자동 업데이트 기본값 | 켜짐 | 꺼짐 | — |

> 📌 **현재 공식 문서 기준**: 예전 튜토리얼의 `/plugin marketplace add anthropics/claude-code` 는 **공식 마켓플레이스가 아니라 데모 마켓플레이스**를 추가한다. `code-review`, `feature-dev`, `commit-commands`, `security-guidance` 처럼 양쪽에 다 있는 플러그인은 **`@claude-plugins-official`** 에서 설치해 중복 설치를 피한다.

**찾아보기** — `/plugin` 의 **Discover** 탭(검색 가능), 웹의 Claude Marketplace, 또는 저장소의 `.claude-plugin/marketplace.json`. Anthropic은 서드파티 마켓플레이스를 검토하지 않으므로 설치 전 내용을 확인한다.

🔗 [Anthropic's marketplaces](https://code.claude.com/docs/en/plugins/anthropic-marketplaces) · [Plugin security and trust](https://code.claude.com/docs/en/plugins/security)

### 설치, 활성화, 비활성화, 제거

| 작업 | 세션 안 | 셸 |
|---|---|---|
| 둘러보기·관리 | `/plugin` (Discover, Installed, Marketplaces, Errors 탭) | `claude plugin list` |
| 설치 | `/plugin install 이름@마켓` → 범위 선택 | `claude plugin install 이름@마켓 --scope project` (기본 user) |
| 끄기/켜기 | Installed 탭에서 `Space` | `claude plugin disable` / `enable 이름@마켓` |
| 제거 | 상세 메뉴 → Uninstall | `claude plugin uninstall 이름@마켓 --scope project` |
| 업데이트 | 상세 메뉴 → Update now | `claude plugin update 이름@마켓` |
| 변경 적용 | `/reload-plugins` | 다음 세션 시작 시 |
| 마켓 추가 | `/plugin marketplace add 소유자/저장소` | `claude plugin marketplace add …` |

- 세션 중에 설치·업데이트해도 **실행 중인 세션은 이전 버전을 유지**한다 → `/reload-plugins` 또는 새 세션.
- `-p` 에서는 `/plugin` 이 동작하지 않는다. 이미 설치된 플러그인은 로드된다.
- 제거하면 `${CLAUDE_PLUGIN_DATA}` 데이터도 기본으로 삭제된다 (`--keep-data` 로 유지).

🔗 [Install and manage plugins](https://code.claude.com/docs/en/plugins/install)

### 팀 워크플로와 자동 설치

**① 저장소 단위** — 프로젝트 `.claude/settings.json` 에 마켓플레이스와 플러그인을 선언한다.

```json
{
  "extraKnownMarketplaces": {
    "acme-plugins": {
      "source": { "source": "github", "repo": "acme/claude-plugins" }
    }
  },
  "enabledPlugins": {
    "team-quality@acme-plugins": true
  }
}
```

| 플러그인 소스 | 팀원 PC에서 |
|---|---|
| 마켓플레이스 안의 **상대 경로** (`"source": "./plugins/team-quality"`) | 폴더를 **신뢰**하면 마켓플레이스가 등록되고 플러그인이 바로 로드됨 |
| 외부 소스 (플러그인 자체 GitHub 저장소 등) | 각자 `claude plugin install 이름@마켓 --scope project` 필요 |

- `extraKnownMarketplaces` 는 **워크스페이스 신뢰 후**에만 적용된다 (신뢰 안 한 폴더에서는 조용히 무시).
- `claude-plugins-official` 플러그인은 `enabledPlugins` 에 넣기만 해도 마켓플레이스가 선언된다.

**② 조직 단위** — 같은 두 키를 **관리형 설정**에 넣으면 모든 PC에 설치된다. CI·`-p` 에서는 백그라운드 설치라 첫 턴에 없을 수 있으므로 `CLAUDE_CODE_SYNC_PLUGIN_INSTALL=1`.

🔗 [Require plugins per repository](https://code.claude.com/docs/en/plugins/org#require-plugins-per-repository) · [Manage plugins for your organization](https://code.claude.com/docs/en/plugins/org)

### 실전 플러그인: 코드 품질

> 🔎 **공식 문서 원칙에서 정리**: 공식 문서의 컴포넌트 예제(리뷰 스킬, 보안 리뷰 에이전트, 편집 후 스크립트 훅)를 Java 팀 구성으로 묶은 예시다.

```text
team-quality/
├── .claude-plugin/plugin.json
├── skills/review/SKILL.md          # /team-quality:review
├── agents/security-reviewer.md     # SQL 인젝션, XSS, 세션 처리 리뷰
├── hooks/hooks.json                # 편집 후 체크 스크립트
└── scripts/check.sh
```

```markdown
---
description: 변경된 Java/JSP 파일을 팀 규칙으로 리뷰한다. 코드 리뷰를 요청받았을 때 사용
---
변경 파일을 읽고 다음 순서로 보고한다.
1. MyBatis 매퍼의 ${} 사용 (SQL 인젝션 위험)
2. JSP 출력 이스케이프 누락
3. 트랜잭션 경계 (@Transactional) 누락
4. 테스트가 없는 서비스 메서드
```

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [ { "type": "command", "command": "\"${CLAUDE_PLUGIN_ROOT}/scripts/check.sh\"" } ]
      }
    ]
  }
}
```

- 에이전트 frontmatter의 `hooks`, `mcpServers` 는 플러그인에서 무시되므로 훅은 `hooks/hooks.json` 에 둔다.
- Java 진단은 직접 만들지 말고 공식 `jdtls-lsp` 를 함께 설치한다.

🔗 [Add each kind of component](https://code.claude.com/docs/en/plugins/components#add-each-kind-of-component)

### 로컬 테스트와 개발 워크플로

**① 마켓플레이스 없이 바로 로드**

```bash
claude --plugin-dir ./team-quality                 # 이 세션에만 로드 (폴더나 .zip)
claude --plugin-url https://example.com/tq.zip     # URL의 zip을 이 세션에만
claude plugin init my-tool                         # ~/.claude/skills/my-tool 에 스캐폴드 → 매 세션 로드
```

**② 수정 → 확인 루프**

1. `claude plugin validate ./team-quality` (`--strict` 면 경고도 실패) → `✔ Validation passed`
2. 세션에서 `/reload-plugins` → `Reloaded:` 줄 확인
3. `/plugin` → Installed 탭(찾은 컴포넌트), **Errors 탭**(로드 실패 사유)
4. 스킬은 `/team-quality:review` 로 실행, MCP는 `/mcp`, 훅은 이벤트를 일으키고 `--debug` 로그 확인

**③ 로컬 마켓플레이스로 설치 흐름까지 확인**

```bash
claude plugin marketplace add ./my-marketplace
claude plugin install team-quality@my-marketplace
```

> 💡 `--plugin-dir` 로 로드한 플러그인은 **같은 이름의 설치된 플러그인을 대체**한다 (개발 중 테스트에 편리, 대신 설치본이 조용히 가려진다).

🔗 [Develop without a marketplace](https://code.claude.com/docs/en/plugins/create#develop-without-a-marketplace) · [Test and debug](https://code.claude.com/docs/en/plugins/create#test-and-debug) · [Name conflicts](https://code.claude.com/docs/en/plugins/loading#name-conflicts)

### 플러그인 캐싱

**① 디스크 구조** (`~/.claude/plugins/`, `CLAUDE_CODE_PLUGIN_CACHE_DIR` 로 변경 가능)

| 경로 | 내용 |
|---|---|
| `cache/<마켓>/<플러그인>/<버전>/` | 설치된 버전별 복사본. `${CLAUDE_PLUGIN_ROOT}` 가 여기를 가리킴 |
| `data/<plugin-id>/` | `${CLAUDE_PLUGIN_DATA}`. 업데이트에도 유지 |
| `marketplaces/<이름>/` | 마켓플레이스 클론 |
| `installed_plugins.json`, `known_marketplaces.json` | 설치·마켓 기록 |

**② 복사 규칙**

- 대부분의 마켓플레이스 플러그인은 설치 시 **캐시로 복사**해서 로드한다. 플러그인 디렉터리 **밖의 파일은 복사되지 않으므로** `../shared` 같은 경로는 동작하지 않는다.
- `--plugin-dir`, 스킬 폴더 플러그인, 로컬 경로 마켓플레이스의 상대 경로 플러그인은 **제자리에서** 로드한다 (수정하면 다음 세션·`/reload-plugins` 에 반영).
- 플러그인 밖을 가리키는 컴포넌트 경로는 거부된다. 경로는 **슬래시**로 쓴다 (백슬래시 경로는 Windows에서만 로드됨).
- 이전 버전 디렉터리는 업데이트·제거 후 **14일** 뒤 정리된다.

**③ 버전과 업데이트**

- 버전 결정 순서: `plugin.json` 의 `version` → 마켓 항목의 `version` → 소스의 커밋 SHA(12자).
- **`version` 을 고정해 두고 커밋만 올리면 업데이트되지 않는다** (`already at the latest version`). 릴리스마다 올리거나, git 마켓이면 `version` 을 빼서 커밋 SHA를 쓰게 한다.
- 자동 업데이트는 첫 메시지 뒤 최대 10분 안의 임의 시점에 돈다. `DISABLE_AUTOUPDATER=1` 등으로 끌 수 있다.

🔗 [Plugin loading reference](https://code.claude.com/docs/en/plugins/loading) · [Versions and updates](https://code.claude.com/docs/en/plugins/loading#versions-and-updates)

### 배포와 마켓플레이스 게시

**① 배포 방법 비교**

| 방법 | 누가 설치 | 자동 업데이트 |
|---|---|---|
| 폴더·zip 전달 | 받은 사람 (`--plugin-dir`) | 없음 |
| **자체 마켓플레이스** (git 저장소의 `.claude-plugin/marketplace.json`) | 저장소에 접근 가능한 사람 (**비공개 저장소면 비공개 마켓**) | 기본 꺼짐 (사용자가 켬) |
| Anthropic 디렉터리 제출 | claude.ai·Cowork에서 추가한 사람 (Claude Code에는 계정 동기화로 `@synced`) | 예 |

**② 자체 마켓플레이스 최소 구성**

```json
{
  "name": "acme-plugins",
  "owner": { "name": "Platform Team" },
  "plugins": [
    { "name": "team-quality", "source": "./plugins/team-quality", "description": "Java/JSP 코드 품질 도구" }
  ]
}
```

팀원 안내: `claude plugin marketplace add acme/claude-plugins` → `claude plugin install team-quality@acme-plugins`

**③ 릴리스 체크리스트 (공식)**

1. 이름은 **영구적**으로 (kebab-case). 이름을 바꾸면 기존 설치가 끊긴다 → 표시 이름은 `displayName`
2. 버전 정책 결정 (`version` 을 매번 올리거나, 빼서 커밋 SHA 사용)
3. `claude plugin validate --strict`
4. 로컬 마켓플레이스에서 설치해 보기
5. `description`, `author`, `homepage`, `repository`, `README.md`

> 📌 **현재 공식 문서 기준**: Anthropic 디렉터리 제출은 claude.ai의 개발자 포털에서 하며 **유료 플랜**이 필요하다. 공식 마켓플레이스(`claude-plugins-official`) 등재는 이 포털로 받지 않는다.

🔗 [Publish and distribute a plugin](https://code.claude.com/docs/en/plugins/publish) · [Create a marketplace](https://code.claude.com/docs/en/plugins/create-marketplace)

---

## 핵심 정리

1. **스크립트·CI는 `claude -p --bare`.** 재현 가능하고, 남의 저장소의 훅·`.mcp.json` 이 신뢰 확인 없이 실행되는 위험을 피한다.
2. **무인 실행은 권한을 미리 정한다**: `--allowedTools`(공백 포함 패턴), `--permission-mode dontAsk`, `--permission-prompts none`, `--max-budget-usd`.
3. **훅 = 지시가 아니라 강제.** 막으려면 **exit 2**(exit 1은 막지 못함), 세밀한 제어는 exit 0 + JSON.
4. **훅은 덮어쓰지 않고 합쳐져서 병렬로 실행**된다. 대화형 세션에서는 워크스페이스 신뢰 후에만 실행된다.
5. **매처는 정확한 문자열 또는 정규식**이고 대소문자를 구분한다. MCP 서버 전체는 `mcp__서버__.*`.
6. **판단이 필요하면 prompt 훅, 조사가 필요하면 agent 훅(실험적)**, 오래 걸리면 `async`. Stop 훅은 `stop_hook_active` 로 무한 반복을 막는다.
7. **MCP는 HTTP 권장(SSE deprecated)**, 범위는 local(기본)·project(`.mcp.json`, 팀 공유)·user. 비밀 값은 `${VAR}`.
8. **MCP 서버는 신뢰할 수 있는 것만**, DB는 읽기 전용 계정. 출력은 기본 25,000토큰 상한.
9. **플러그인 = 스킬·에이전트·훅·MCP·LSP 등을 묶은 배포 단위.** 범위는 local > project > user, project로 켜도 외부 소스 플러그인은 각자 설치해야 한다.
10. **Java는 `jdtls` 설치 후 `jdtls-lsp@claude-plugins-official`** 로 편집 후 진단을 받는다. 개발은 `--plugin-dir`, `claude plugin validate`, `/reload-plugins` 순서로.

## 실무에서 바로 써먹기

**상황**: Windows + IntelliJ, Spring + MyBatis + JSP Maven 프로젝트. 팀에 공통 안전장치와 리뷰 도구를 배포하고, 야간 CI에서 자동 점검을 돌린다. (경로·이름은 예시)

```text
[1. 반드시 지킬 규칙은 훅으로]
  .claude/hooks/protect-files.ps1 : .env, application-prod*, WEB-INF/web.xml 수정 시 exit 2
  .claude/settings.json → PreToolUse, matcher "Edit|Write", shell "powershell"
  모의 JSON을 파이프로 넣어 exit 코드 확인 → /hooks 에 보이는지 확인 → 커밋

[2. 알림]
  ~/.claude/settings.json → Notification 훅 (PowerShell MessageBox), matcher "permission_prompt"

[3. Java 코드 인텔리전스]
  jdtls 설치 → Get-Command jdtls 로 PATH 확인
  /plugin install jdtls-lsp@claude-plugins-official → 일부러 타입 에러를 내서 진단 확인

[4. 팀 도구는 플러그인으로]
  team-quality 플러그인 (review 스킬 + security-reviewer 에이전트 + 편집 후 체크 훅)
  claude --plugin-dir ./team-quality 로 테스트 → claude plugin validate --strict
  사내 git 저장소에 marketplace.json 으로 게시 (비공개 저장소 = 비공개 마켓)
  프로젝트 .claude/settings.json 에 extraKnownMarketplaces + enabledPlugins → 커밋

[5. 사내 MCP]
  .mcp.json 에 사내 이슈 트래커 HTTP 서버, 토큰은 ${ISSUE_TOKEN}
  DB 조회 서버는 읽기 전용 계정 DSN, 각자 local 범위로만 추가

[6. 야간 CI 점검 (bare 모드)]
  claude --bare -p "src/main/resources/mapper 에서 ${} 사용 위치를 찾아줘" \
    --allowedTools "Read,Grep,Glob" --permission-mode dontAsk \
    --output-format json --json-schema '<files 배열 스키마>' --max-budget-usd 1
  → structured_output 을 리포트로 저장, 종료 코드로 실패 판정
```

## 참고한 공식 문서

- [Run Claude Code programmatically](https://code.claude.com/docs/en/headless) · [CLI reference](https://code.claude.com/docs/en/cli-reference)
- [Automate actions with hooks](https://code.claude.com/docs/en/hooks-guide) · [Hooks reference](https://code.claude.com/docs/en/hooks)
- [Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp)
- [Plugins overview](https://code.claude.com/docs/en/plugins/overview) · [Install and manage plugins](https://code.claude.com/docs/en/plugins/install) · [Create a plugin](https://code.claude.com/docs/en/plugins/create) · [Add components](https://code.claude.com/docs/en/plugins/components)
- [Plugin loading reference](https://code.claude.com/docs/en/plugins/loading) · [Code intelligence plugins](https://code.claude.com/docs/en/plugins/code-intelligence) · [Anthropic's marketplaces](https://code.claude.com/docs/en/plugins/anthropic-marketplaces)
- [Manage plugins for your organization](https://code.claude.com/docs/en/plugins/org) · [Create a marketplace](https://code.claude.com/docs/en/plugins/create-marketplace) · [Publish and distribute a plugin](https://code.claude.com/docs/en/plugins/publish)
