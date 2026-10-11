# 📒 5장. 출력 인터페이스

> **기준**: 「클로드코드로 시작하는 실전 에이전틱 코딩」 CHAPTER 05 목차 + Claude Code 공식 문서([code.claude.com/docs](https://code.claude.com/docs), 2026-10-11 확인). 책과 달라진 부분은 **📌 현재 공식 문서 기준**, 공식 문서에 전용 설명이 없는 부분은 **🔎 공식 문서 원칙에서 정리** 로 표시했다.
>
> 관련 노트: [1장](./01-getting-started.md) (컨텍스트, 체크포인트) · [2장](./02-workflow-and-settings.md) (settings.json, 우선순위) · [3장](./03-agent-skills.md) (스킬) · [6장](./06-memory-and-sessions.md) (CLAUDE.md) · [7장](./07-automation.md) (훅)

---

## 5-1. 출력 스타일 커스터마이징

### 출력 스타일의 핵심 개념

**① 개념** — 출력 스타일(output style)은 **세션의 모든 응답**에 적용되는 지시 묶음이다. Claude의 역할·말투·응답 형식을 바꾼다.

**② 왜** — "짧게 답해줘", "왜 그렇게 했는지 설명해줘"를 매 프롬프트마다 반복하지 않아도 된다. 코딩이 아닌 역할(문서 작성 도우미 등)로 바꿀 수도 있다.

**③ 동작 원리 (공식 문서)**

- 활성 스타일의 지시는 **매 요청마다** 함께 전송된다.
- 커스텀 스타일은 기본적으로 Claude Code의 **소프트웨어 엔지니어링 지시(변경 범위 정하기, 주석 작성, 검증 방법 등)를 뺀다.** 유지하려면 `keep-coding-instructions: true`.
- 메인 대화와 **fork**에는 적용되지만, 일반 **서브에이전트에는 적용되지 않는다** (자기 시스템 프롬프트로 실행되기 때문).
- 스타일 지시는 입력 토큰을 늘린다. 첫 요청 뒤에는 프롬프트 캐싱으로 비용이 줄어든다.

**④ 한계** — 출력 스타일은 **"지시"일 뿐 강제가 아니다.** 반드시 일어나야 하는 일(편집 후 포맷, 명령 차단)은 훅으로 한다.

🔗 [Output styles](https://code.claude.com/docs/en/output-styles) · [How output styles work](https://code.claude.com/docs/en/output-styles#how-output-styles-work)

### 내장 출력 스타일

> 📌 **현재 공식 문서 기준**: 내장 스타일은 Default 외에 **Proactive, Concise, Explanatory, Learning 4가지**다 (Concise는 v2.1.237 이상). 책의 스타일 목록과 다르면 공식 목록을 기준으로 삼는다.

| 스타일 | 바뀌는 점 | 이럴 때 |
|---|---|---|
| **Default** | 스타일 지시 없음. 표준 시스템 프롬프트로 동작 | 기본 |
| **Proactive** | 바로 구현을 시작하고, 일상적인 결정은 합리적으로 가정한다. 요청하지 않으면 plan 모드로 가지 않음 | 일상적 결정은 맡기고, 틀리면 내가 방향을 바꾸겠다 |
| **Concise** | 결과부터 말하고 서론·진행 중계·마무리 요약을 뺀다. 단순 질문은 1–3문장 | 기본 응답이 너무 길다 |
| **Explanatory** | 작업은 Default와 같고 `★ Insight` 블록으로 선택 이유를 설명 (파일에 주석으로 쓰지 않음) | 낯선 코드베이스를 익히거나 근거가 필요할 때 |
| **Learning** | Insight에 더해 설계 판단이 필요한 부분을 `TODO(human)` 으로 남기고 **직접 작성하게 한다** | 작업을 진행하면서 손으로 연습하고 싶을 때 |

**알아둘 점**

- **Proactive는 권한 모드를 바꾸지 않는다.** 권한 프롬프트는 그대로 뜬다. 데이터 삭제나 공유·운영 시스템 변경 전에는 대화로 확인하라는 지시가 들어 있지만, 권한 프롬프트와는 별개다.
- **Concise여도 생략하지 않는 것**: 사용자가 요청한 설명, 에러 보고, 실패한 테스트 출력, 보안 경고, 파괴적 작업 확인.
- Explanatory·Learning은 설계상 응답이 길어져 **출력 토큰이 늘어난다.**

🔗 [Built-in output styles](https://code.claude.com/docs/en/output-styles#built-in-output-styles)

### 출력 스타일 변경 방법

| 방법 | 동작 |
|---|---|
| `/output-style concise` | 바로 전환 (대소문자 무시). 인수 없이 실행하면 목록과 현재 스타일 표시 |
| `/config` → **Output style** | 메뉴에서 선택 |
| VS Code 확장 | `/` 명령 메뉴 → **Output styles** (커스텀 포함) |
| 설정 파일 `outputStyle` | `"outputStyle": "Explanatory"` — **대소문자 구분** |

```json
{
  "outputStyle": "Explanatory"
}
```

**알아둘 점**

- 명령과 메뉴로 고르면 **`.claude/settings.local.json`(로컬 프로젝트)** 에 저장된다.
- 모든 프로젝트 기본값으로 쓰려면 `~/.claude/settings.json` 에 `outputStyle` 을 넣는다. 프로젝트 설정이 있으면 프로젝트가 이긴다 (우선순위는 [2장](./02-workflow-and-settings.md) 참고).
- 설정 파일 값은 **이름이 정확히 일치해야** 한다. `explanatory`(소문자)처럼 틀리면 조용히 Default가 된다.
- 세션 중에 바꾸면 **다음 메시지부터** 적용된다.

> 📌 **현재 공식 문서 기준**: v2.1.251 이전에는 스타일을 바꾸면 `/clear` 나 새 세션 뒤에야 적용됐다. 지금은 다음 메시지부터 바로 적용된다. `/output-style` 은 v2.1.269부터 비대화형(`-p`)에서도 쓸 수 있다.

🔗 [Change your output style](https://code.claude.com/docs/en/output-styles#change-your-output-style)

### 커스텀 출력 스타일 생성

**① 위치** — 파일 이름이 스타일 이름이 된다 (frontmatter `name` 으로 바꿀 수 있음).

| 수준 | 경로 |
|---|---|
| 사용자 | `~/.claude/output-styles/` |
| 프로젝트 | `.claude/output-styles/` (작업 디렉터리부터 저장소 루트까지 모두 읽고, 같은 이름이면 가까운 쪽) |
| 관리형 정책 | 관리형 설정 디렉터리 안의 `.claude/output-styles/` |
| 플러그인 | 플러그인의 `output-styles/` |

**② frontmatter**

| 필드 | 설명 |
|---|---|
| `name` | `/config` 선택 목록에 보일 이름 (기본: 파일 이름) |
| `description` | 선택 목록의 설명 |
| `keep-coding-instructions` | `true` 면 Claude Code의 엔지니어링 지시를 유지. 기본 `false` |
| `force-for-plugin` | 플러그인 스타일 전용. `true` 면 플러그인이 켜질 때 자동 적용 (사용자 `outputStyle` 보다 우선) |

**③ 예시** — 말투만 바꾸고 코딩 방식은 유지하는 팀 리뷰 스타일 (파일: `.claude/output-styles/korean-review.md`)

```markdown
---
name: Korean review
description: 한국어로, 변경 요약과 위험 요소를 먼저 보고한다
keep-coding-instructions: true
---
모든 응답은 한국어로 작성한다. 코드, 명령, 파일 경로, 에러 메시지는 원문 그대로 둔다.

## 응답 순서
1. 무엇을 바꿨는지 한두 문장
2. 영향 받는 파일 목록
3. 위험 요소 (트랜잭션, 세션, 인코딩, SQL)
4. 내가 직접 확인해야 할 것
```

**④ 주의**

- 터미널에서는 **시작할 때 스타일 파일을 읽는다.** 세션 중에 만들거나 고쳤다면 Claude Code를 재시작한다.
- 필드 이름 오타는 **에러 없이 무시**된다. YAML 파싱이 실패하면 필드 없이 파일 이름으로 로드되니, `claude --debug` 로 파싱 에러를 확인한다.

🔗 [Create a custom output style](https://code.claude.com/docs/en/output-styles#create-a-custom-output-style) · [Frontmatter reference](https://code.claude.com/docs/en/output-styles#frontmatter)

### 관련 기능과 비교

| 원하는 것 | 쓸 기능 | 이유 |
|---|---|---|
| 모든 응답의 말투·길이·형식, 다른 역할 | **출력 스타일** | 세션 전체에 적용, 명령 하나로 전환 |
| 프로젝트 규칙·명령·구조를 알게 하기 | **`CLAUDE.md`** ([6장](./06-memory-and-sessions.md)) | 코드베이스 지식. 어떤 스타일을 골라도 로드됨 |
| 특정 작업 절차 (릴리스 체크리스트 등) | **스킬** ([3장](./03-agent-skills.md)) | 필요할 때만 로드 |
| 예외 없이 매번 일어나야 하는 일 | **훅** ([7장](./07-automation.md)) | Claude의 판단과 무관하게 Claude Code가 실행 |
| 별도 지시·모델·도구를 가진 도우미 | **서브에이전트** ([4장](./04-subagents.md)) | 별도 컨텍스트에서 실행하고 요약만 반환 |
| 실행할 때 지시를 덧붙이기 | **`--append-system-prompt`** | 기본 시스템 프롬프트를 지우지 않고 추가 |

조합해서 쓴다: **`CLAUDE.md` = 무엇을 알아야 하나, 출력 스타일 = 어떻게 답하나, 훅 = 반드시 지킬 것.**

🔗 [Choose between an output style and other features](https://code.claude.com/docs/en/output-styles#choose-between-an-output-style-and-other-features)

### 팀 표준 출력 스타일 구성

> 🔎 **공식 문서 원칙에서 정리**: "팀 표준 스타일" 전용 절은 없다. 공식 문서의 저장 위치·우선순위 규칙을 조합한 운영 방법이다.

- **스타일 파일은 공유, 선택은 개인**: `.claude/output-styles/` 를 커밋해 팀원이 같은 스타일을 고를 수 있게 한다. `/output-style` 로 고르면 각자의 `settings.local.json` 에 저장되므로 팀원끼리 충돌하지 않는다.
- **기본값까지 통일하려면**: 프로젝트 `.claude/settings.json` 에 `outputStyle` 을 넣는다. 단, 개인이 로컬 설정에서 바꾸면 개인 설정이 우선한다.
- **조직 전체 강제가 필요하면**: 관리형 정책 위치에 두거나, 플러그인 스타일에 `force-for-plugin: true` 를 쓴다.
- **(제안)** 팀 스타일에는 `keep-coding-instructions: true` 를 기본으로 둔다. 빼면 변경 범위·검증 같은 엔지니어링 지시가 사라져 코딩 품질이 달라질 수 있다.

🔗 [Settings precedence](https://code.claude.com/docs/en/settings#settings-precedence)

### 스타일 검증과 다국어 출력

> 🔎 **공식 문서 원칙에서 정리**: 스타일 "검증"과 "다국어" 전용 기능은 없다. 아래는 공식 문서의 디버깅·설정 수단으로 확인하는 방법이다.

**검증 체크리스트**

1. `/output-style` (인수 없음) → 현재 스타일이 표시되는지
2. 목록에 커스텀 스타일이 없다 → 경로 확인, **Claude Code 재시작**
3. 필드가 안 먹는다 → 필드 이름 오타, `claude --debug` 로 YAML 파싱 에러 확인
4. 설정 파일로 지정했는데 Default다 → 이름 대소문자 확인
5. 상태 표시줄 스크립트는 `output_style.name` 필드를 받으므로, 현재 스타일을 화면 하단에 띄워 둘 수 있다 (5-2 참고)

**다국어 출력**

- 응답 언어는 스타일 지시에 "한국어로 답한다"처럼 쓰면 된다 (위 예시).
- 스타일과 별개로 설정에 `language` 를 두는 방법도 있다. 2장 "주요 설정 옵션" 참고.
- **(제안)** "코드·명령·로그·에러 메시지는 번역하지 않는다"를 함께 써 둔다. 에러 메시지를 번역하면 검색과 대조가 어려워진다.

🔗 [Debug your configuration](https://code.claude.com/docs/en/debug-your-config) · [Settings](https://code.claude.com/docs/en/settings)

### 환각 방지 지침

> 🔎 **공식 문서 원칙에서 정리**: 공식 문서에 "환각 방지 스타일"은 없다. 공식 문서가 말하는 원칙은 **스타일은 지시일 뿐 보장하지 않는다**, **확실해야 하는 것은 훅으로**, **Claude가 스스로 검증할 수단(테스트 등)을 준다**이다.

**스타일에 넣을 만한 지시 (제안)**

```markdown
## 근거 규칙
- 파일 내용, 클래스, 메서드, 설정 값은 직접 읽은 뒤에만 언급한다. 읽지 않았으면 "확인하지 않음"이라고 쓴다.
- 존재를 확인하지 않은 API, 라이브러리 버전, 설정 키를 지어내지 않는다.
- "테스트 통과", "빌드 성공"은 실제로 실행한 결과가 있을 때만 쓴다.
- 추측이 섞이면 추측이라고 표시한다.
```

**지시만으로 부족할 때**

- 빌드·테스트 실행으로 검증하게 한다 (2장 "탐색 → 계획 → 구현 → 커밋").
- 반드시 막아야 하는 것은 훅으로 강제한다 ([7장](./07-automation.md)).
- 커스텀 스타일에서 `keep-coding-instructions` 를 빼면 Claude Code의 "작업 검증" 지시도 함께 빠진다는 점을 기억한다.

🔗 [Output styles](https://code.claude.com/docs/en/output-styles) · [Best practices](https://code.claude.com/docs/en/best-practices)

---

## 5-2. 상태 표시줄 (Status Line)

### 상태 표시줄 설정 방법

**① 개념** — 화면 하단에 내가 설정한 **셸 명령의 출력**을 계속 보여 주는 줄이다. 컨텍스트 사용률, 비용, git 브랜치 등을 한눈에 본다. 내장 하단 배지를 대체하지 않고 그 위 별도 줄에 표시된다.

**② 왜** — 컨텍스트가 얼마나 찼는지 계속 보여야 `/compact`·`/clear` 시점을 판단할 수 있다 ([1장](./01-getting-started.md) 컨텍스트 관리). 세션을 여러 개 쓸 때 어느 세션인지 구분하기도 쉽다.

**③ 설정 방법 2가지**

```text
/statusline show model name and context percentage with a progress bar
```

`/statusline` 에 자연어로 요청하면 `~/.claude/` 에 스크립트를 만들고 설정까지 고쳐 준다. 직접 설정하려면 `settings.json` 에 `statusLine` 을 넣는다.

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/statusline.sh",
    "padding": 2
  }
}
```

| 필드 | 설명 |
|---|---|
| `type` | `"command"` |
| `command` | 스크립트 경로 또는 인라인 셸 명령 |
| `padding` | 추가 가로 여백 (기본 0) |
| `refreshInterval` | N초마다 추가로 다시 실행 (최소 1). 시계처럼 시간 기반 정보를 보여 줄 때 |
| `hideVimModeIndicator` | 스크립트가 vim 모드를 직접 표시할 때 기본 `-- INSERT --` 숨김 |

**④ 끄기** — `/statusline delete`(또는 clear, remove) 또는 설정에서 `statusLine` 삭제.

> 📌 **현재 공식 문서 기준**: 사용자 정의 상태 표시줄을 켜면 `esc to interrupt` 같은 하단 키 힌트 대부분이 표시되지 않는다. 서브에이전트 패널의 각 행을 바꾸는 `subagentStatusLine` 설정도 따로 있다.

🔗 [Set up a status line](https://code.claude.com/docs/en/statusline#set-up-a-status-line)

### 스크립트 작동 원리

**① 데이터 흐름** — Claude Code가 세션 정보를 **JSON으로 stdin에 넣고**, 스크립트가 **stdout에 출력한 내용**을 표시한다.

**② 언제 다시 실행되나**

- 세션 시작·재개 시 1회
- 새 assistant 메시지, `/compact` 완료, 권한 모드 변경, vim 모드 전환, `command` 변경
- `refreshInterval` 타이머, rate limit 창 리셋 시각, 프롬프트 캐시 만료 시각
- **300ms 디바운스** — 연속 변경은 모아서 한 번 실행. 실행 중에 새 업데이트가 오면 **진행 중인 스크립트는 취소**된다.

**③ 주요 입력 필드**

| 필드 | 내용 |
|---|---|
| `model.display_name` | 현재 모델 표시 이름 |
| `workspace.current_dir`, `workspace.project_dir` | 현재 디렉터리, 시작 디렉터리 |
| `context_window.used_percentage` | 컨텍스트 사용률 (미리 계산된 값) |
| `context_window.context_window_size` | 최대 컨텍스트 크기 (기본 200000, 확장 모델은 1000000) |
| `cost.total_cost_usd` | 세션 예상 비용 (정가 기준 클라이언트 계산, 실제 청구와 다를 수 있음) |
| `cost.total_lines_added`, `cost.total_lines_removed` | 변경 라인 수 |
| `rate_limits.five_hour.used_percentage` | 5시간 사용 한도 소진율 |
| `session_id`, `session_name`, `transcript_path` | 세션 식별 정보 |
| `output_style.name` | 현재 출력 스타일 |
| `effort.level`, `thinking.enabled`, `fast_mode` | 추론 수준, 확장 사고, fast 모드 |
| `pr.number`, `pr.review_state` | 현재 브랜치의 열린 PR |
| `worktree.*`, `agent.name`, `vim.mode` | 워크트리, `--agent` 이름, vim 모드 |

**④ 출력할 수 있는 것** — 여러 줄(`echo` 한 번 = 한 줄), ANSI 색상, OSC 8 클릭 가능한 링크. 스크립트 안에서 `tput cols` 로는 터미널 폭을 못 읽으므로 환경 변수 `COLUMNS` 를 쓴다.

🔗 [How status lines work](https://code.claude.com/docs/en/statusline#how-status-lines-work) · [Available data](https://code.claude.com/docs/en/statusline#available-data)

### Bash 스크립트 예시

> 📌 **현재 공식 문서 기준**: 공식 예제는 컨텍스트 사용률, 색상 git 상태, 비용·시간, 여러 줄, 클릭 링크, rate limit, 캐싱 7가지다. 아래는 그중 기본형이다. Bash 예제는 `jq` 가 필요하다.

**기본형** — 모델, 폴더, 컨텍스트 사용률 (`~/.claude/statusline.sh`)

```bash
#!/bin/bash
input=$(cat)
MODEL=$(echo "$input" | jq -r '.model.display_name')
DIR=$(echo "$input" | jq -r '.workspace.current_dir')
# "// 0" 은 값이 null일 때의 기본값
PCT=$(echo "$input" | jq -r '.context_window.used_percentage // 0' | cut -d. -f1)
echo "[$MODEL] 📁 ${DIR##*/} | ${PCT}% context"
```

```bash
chmod +x ~/.claude/statusline.sh
```

**인라인 명령** — 스크립트 파일 없이 `command` 에 바로 쓸 수도 있다.

```json
{
  "statusLine": {
    "type": "command",
    "command": "jq -r '\"[\\(.model.display_name)] \\(.context_window.used_percentage // 0)% context\"'"
  }
}
```

🔗 [Build a status line step by step](https://code.claude.com/docs/en/statusline#build-a-status-line-step-by-step) · [Examples](https://code.claude.com/docs/en/statusline#examples)

### Windows에서 설정하기 (PowerShell, Git Bash)

> 📌 **현재 공식 문서 기준**: Windows에서는 **Git Bash가 있으면 Git Bash로, 없으면 PowerShell로** 상태 표시줄 명령을 실행한다. Git Bash는 따옴표 없는 백슬래시를 이스케이프 문자로 처리하므로, `command` 경로는 **슬래시(`/`)로** 쓴다. 그렇지 않으면 화면에 에러 없이 실패한다.

**PowerShell 스크립트** — 어느 셸로 실행되든 `powershell` 로 호출하면 동작한다.

```json
{
  "statusLine": {
    "type": "command",
    "command": "powershell -NoProfile -File C:/Users/username/.claude/statusline.ps1"
  }
}
```

```powershell
# C:\Users\username\.claude\statusline.ps1
$input_json = $input | Out-String | ConvertFrom-Json
$cwd = $input_json.cwd
$model = $input_json.model.display_name
$used = $input_json.context_window.used_percentage
$dirname = Split-Path $cwd -Leaf
if ($used) {
    Write-Host "$dirname [$model] ctx: $used%"
} else {
    Write-Host "$dirname [$model]"
}
```

**Git Bash에 `jq` 가 없을 때** — 공식 예시는 `grep` 과 `cut` 으로 JSON을 파싱한다.

```bash
#!/usr/bin/env bash
input=$(cat)
cwd=$(echo "$input" | grep -o '"cwd":"[^"]*"' | cut -d'"' -f4)
model=$(echo "$input" | grep -o '"display_name":"[^"]*"' | cut -d'"' -f4)
dirname="${cwd##*[/\\]}"
echo "$dirname [$model]"
```

🔗 [Windows configuration](https://code.claude.com/docs/en/statusline#windows-configuration)

### Python, Node.js 스크립트

공식 예제는 모두 **Bash, Python, Node.js 3가지 버전**으로 제공된다. 받는 JSON과 출력 규칙은 같다.

```python
#!/usr/bin/env python3
import json, sys, os
data = json.load(sys.stdin)
model = data['model']['display_name']
directory = os.path.basename(data['workspace']['current_dir'])
pct = int(data.get('context_window', {}).get('used_percentage') or 0)
print(f"[{model}] 📁 {directory} | {pct}% context")
```

```javascript
#!/usr/bin/env node
let input = '';
process.stdin.on('data', chunk => input += chunk);
process.stdin.on('end', () => {
  const data = JSON.parse(input);
  const pct = Math.floor(data.context_window?.used_percentage || 0);
  console.log(`[${data.model.display_name}] 📁 ${require('path').basename(data.workspace.current_dir)} | ${pct}% context`);
});
```

Windows에서는 `"command": "python C:/Users/username/.claude/statusline.py"` 처럼 인터프리터를 명시하면 실행 권한 문제가 없다.

> 🔎 **공식 문서 원칙에서 정리**: 위 Python·Node.js 코드는 공식 기본형 Bash 예제를 같은 필드로 옮긴 것이다. 공식 원문 예제는 아래 "성능 최적화"의 캐싱 예제처럼 언어별로 제공된다.

🔗 [Examples](https://code.claude.com/docs/en/statusline#examples)

### 헬퍼 함수 모듈화와 ANSI 코드

**ANSI 색상** — `\033[32m`(초록), `\033[33m`(노랑), `\033[31m`(빨강), `\033[0m`(초기화). 터미널이 지원해야 한다.

**OSC 8 링크** — 텍스트를 클릭 가능하게 만든다 (Windows는 `Ctrl+클릭`). iTerm2, Kitty, WezTerm 같은 지원 터미널이 필요하다. `echo -e` 보다 `printf '%b'` 가 안정적이다.

> 🔎 **공식 문서 원칙에서 정리**: "헬퍼 함수 모듈화"는 공식 문서 용어가 아니다. 공식 예제의 색상 분기·캐시 검사 로직을 함수로 묶어 재사용하는 방식으로 정리했다.

```bash
#!/bin/bash
# 색상 헬퍼: 사용률에 따라 초록/노랑/빨강
color_for_pct() {
  local p=$1
  if [ "$p" -ge 80 ]; then printf '\033[31m'
  elif [ "$p" -ge 50 ]; then printf '\033[33m'
  else printf '\033[32m'; fi
}
RESET='\033[0m'

input=$(cat)
PCT=$(echo "$input" | jq -r '.context_window.used_percentage // 0' | cut -d. -f1)
MODEL=$(echo "$input" | jq -r '.model.display_name')
printf '%b\n' "[$MODEL] $(color_for_pct "$PCT")${PCT}%${RESET} context"
```

**(제안)** 여러 스크립트에서 쓰는 함수는 `~/.claude/statusline-lib.sh` 같은 파일로 빼고 `source` 한다.

> ⚠️ 색상·링크 같은 이스케이프 시퀀스는 다른 UI 갱신과 겹치면 화면이 깨질 수 있다. 여러 줄 + 이스케이프 조합이 특히 그렇다. 깨지면 일반 텍스트로 단순화한다.

🔗 [What your script can output](https://code.claude.com/docs/en/statusline#what-your-script-can-output) · [Git status with colors](https://code.claude.com/docs/en/statusline#git-status-with-colors) · [Clickable links](https://code.claude.com/docs/en/statusline#clickable-links)

### 성능 최적화

**① 왜** — 스크립트는 활성 세션 동안 자주 실행된다. 큰 저장소에서 `git status`, `git diff` 는 느려서 화면 갱신을 늦춘다. 느린 스크립트가 끝나기 전에 다음 갱신이 오면 취소되어 **오래된 출력**이 남는다.

**② 공식 방법: 결과 캐싱** — git 정보를 임시 파일에 저장하고 **5초**가 지났을 때만 다시 계산한다.

- 캐시 파일 이름은 **세션 안에서는 고정, 세션끼리는 달라야** 한다 → `session_id` 를 이름에 넣는다.
- `$$`, `os.getpid()` 같은 프로세스 ID는 매번 바뀌어 캐시가 무의미해진다.

```bash
SESSION_ID=$(echo "$input" | jq -r '.session_id')
CACHE_FILE="/tmp/statusline-git-cache-$SESSION_ID"
CACHE_MAX_AGE=5  # 초
# 캐시가 없거나 5초보다 오래됐을 때만 git 명령 실행 → 결과를 "브랜치|staged|modified" 로 저장
```

**③ 그 밖에**

- 출력은 짧게 유지 (폭이 좁으면 잘리거나 줄바꿈된다).
- 시간 기반 정보가 필요할 때만 `refreshInterval` 을 쓴다.

🔗 [Cache expensive operations](https://code.claude.com/docs/en/statusline#cache-expensive-operations) · [Tips](https://code.claude.com/docs/en/statusline#tips)

### 문제 해결

| 증상 | 확인할 것 |
|---|---|
| 아무것도 안 보인다 | 실행 권한(`chmod +x`), stdout으로 출력하는지, 직접 실행해 보기, Windows는 경로를 `/` 로 |
| 폴더를 신뢰하기 전에는 비어 있다 | 상태 표시줄도 셸 명령이라 **워크스페이스 신뢰** 후에만 실행된다. `claude --debug` 로그에 `Status line command skipped: workspace trust not accepted` |
| 갑자기 사라졌다 | `disableAllHooks: true` 또는 조직의 `allowManagedHooksOnly` 정책 → 관리형 설정의 상태 표시줄만 동작 |
| `--` 나 빈 값 | 첫 API 응답 전에는 필드가 `null` 일 수 있다 → `// 0` 같은 기본값 처리 |
| `/context` 와 사용률이 다르다 | 상태 표시줄은 마지막 API 응답 기준, `/context` 는 그 뒤 추가된 메시지 추정치를 더한다 |
| 링크가 클릭되지 않는다 | 터미널 지원 여부, PowerShell에서 `$env:FORCE_HYPERLINK = "1"; claude` |
| 스크립트 에러 | **exit 0으로 끝나야** 표시된다. 0이 아니거나 출력이 없으면 빈 줄 |

**모의 입력으로 테스트하기**

```bash
echo '{"model":{"display_name":"Opus"},"workspace":{"current_dir":"/home/user/project"},"context_window":{"used_percentage":25},"session_id":"test-session-abc"}' | ./statusline.sh
```

`claude --debug` 로 실행하면 상태 표시줄 스크립트의 stderr와 첫 실행의 종료 코드가 로그에 남는다.

🔗 [Troubleshooting](https://code.claude.com/docs/en/statusline#troubleshooting)

---

## 5-3. 대화형 모드 활용

### 고급 제어 기능

> 기본 단축키(`Esc`, `Esc` 두 번, `Shift+Tab`)와 대화형/비대화형 비교는 [1장 CLI 기초](./01-getting-started.md)에 정리했다. 여기서는 그 밖의 제어 기능만 다룬다.

**① 작업 중에 메시지 대기열에 넣기**

- Claude가 일하는 중에 입력하고 `Enter` → 중단하지 않고 **대기열**에 넣는다.
- 도구 호출 중이면 그 호출이 끝나는 대로 **같은 턴 안에서** 전달된다. `/명령`, `!명령` 은 턴이 끝난 뒤 순서대로 실행된다.
- `Ctrl+Enter`(또는 `Ctrl+X Ctrl+S`): 대기열을 지금 보낸다 (v2.1.275+).
- 첫 줄에서 `Up`: 대기열을 입력창으로 되돌려 수정한다.

**② 백그라운드 작업 (`Ctrl+B`)**

- 실행 중인 Bash 명령이나 에이전트를 백그라운드로 보내고 대화를 계속한다. 출력은 파일에 기록되고 Claude가 나중에 읽는다.
- 빌드 도구, 테스트 러너, 개발 서버에 유용하다. `/tasks` 로 실행 중인 셸과 서브에이전트를 본다.
- 명령이 타임아웃에 걸리면 중단 대신 **자동으로 백그라운드로** 간다.

**③ `!` 셸 모드** — `! git status` 처럼 입력하면 Claude를 거치지 않고 바로 실행하고, 출력이 세션에 추가된다.

**④ `/btw` 곁가지 질문**

- 대화 기록에 남기지 않고 질문한다. Claude가 작업 중이어도 쓸 수 있다.
- **도구가 없다**: 이미 컨텍스트에 있는 내용으로만 답한다 (파일 읽기·명령 실행 불가).
- 이어서 도구가 필요하면 답변 화면에서 `f` → fork 서브에이전트로 이어 간다.

**⑤ `/diff`** — 작업 트리 변경을 Claude Code 안에서 본다. 턴별(`T1`, `T2`…) 변경도 따로 볼 수 있다. 전체 화면 렌더링에서는 옆 패널로 열린다.

**⑥ 작업 목록 (`Ctrl+T`)** — Claude가 만든 할 일 체크리스트를 표시하거나 숨긴다. 압축 후에도 유지된다.

**⑦ 그 밖에**

- `Ctrl+G`: 프롬프트를 외부 편집기에서 작성
- `Ctrl+S`: 작성 중인 프롬프트 잠시 보관(stash) / 복원
- `Ctrl+R`: 입력 기록 역방향 검색
- `Ctrl+O`: 상세 기록(transcript) 보기 — 도구 호출 상세, 접힌 진단 메시지 펼치기
- **vim 편집 모드**: `/config` → Editor mode (📌 `/vim` 명령은 v2.1.92에서 제거됨)

> 📌 **현재 공식 문서 기준**: Windows에서 Node·Bun 런타임이 VT 입력 모드를 켜지 않으면 권한 모드 전환이 `Shift+Tab` 대신 **`Alt+M`** 이다. 모델 전환은 `Alt+P`, 확장 사고 토글은 `Alt+T` 인데, Opus 5.5·Sonnet 5.5처럼 항상 확장 사고를 쓰는 모델에서는 효과가 없다.

🔗 [Interactive mode](https://code.claude.com/docs/en/interactive-mode) · [Queue messages while Claude works](https://code.claude.com/docs/en/interactive-mode#queue-messages-while-claude-works) · [Background Bash commands](https://code.claude.com/docs/en/interactive-mode#background-bash-commands) · [Side questions with /btw](https://code.claude.com/docs/en/interactive-mode#side-questions-with-%2Fbtw)

### 단축키 빠른 참조

**일반 제어**

| 단축키 | 동작 |
|---|---|
| `Ctrl+C` | 실행 중단. 아무것도 실행 중이 아니면 입력 지우기, 두 번이면 종료 |
| `Ctrl+D` | 세션 종료 (800ms 안에 두 번) |
| `Esc` | Claude 중단 (지금까지 한 작업은 유지) |
| `Esc` `Esc` | 입력이 있으면 지우기, 비어 있으면 되감기 메뉴 |
| `Shift+Tab` (Windows 일부: `Alt+M`) | 권한 모드 순환 |
| `Alt+P` / `Alt+T` / `Alt+O` | 모델 전환 / 확장 사고 / fast 모드 (Windows·Linux) |
| `Ctrl+B` | 실행 중인 작업을 백그라운드로 |
| `Ctrl+T` | 작업 체크리스트 표시/숨김 |
| `Ctrl+O` | 상세 기록 보기 |
| `Ctrl+G` | 외부 편집기로 프롬프트 작성 |
| `Ctrl+R` | 입력 기록 검색 |
| `Ctrl+S` | 프롬프트 보관/복원 |
| `Ctrl+L` | 화면 다시 그리기 (화면이 깨졌을 때) |
| `Ctrl+V`, Windows는 `Alt+V` | 클립보드 이미지 붙여넣기 |
| `Ctrl+Enter` | 대기열 메시지 즉시 전송 |

**여러 줄 입력**

| 방법 | 비고 |
|---|---|
| `\` 다음 `Enter` | 모든 터미널 |
| `Shift+Enter` | Windows Terminal, iTerm2, WezTerm 등에서 기본 지원 |
| `Ctrl+J` | 설정 없이 모든 터미널 |

**입력 접두어**

| 입력 | 동작 |
|---|---|
| `/` (맨 앞) | 명령 또는 스킬 |
| `!` (맨 앞) | 셸 모드 |
| `@` | 파일 경로 자동 완성 |
| `?` (빈 입력) | 단축키 도움말 |

**텍스트 편집** — `Ctrl+A`/`Ctrl+E` 줄 처음/끝, `Ctrl+K` 줄 끝까지 삭제, `Ctrl+U` 줄 처음까지 삭제, `Ctrl+W` 앞 단어 삭제, `Ctrl+Y` 삭제한 텍스트 붙여넣기, `Ctrl+_` 입력 편집 취소.

> 💡 단축키는 `/keybindings` 로 `~/.claude/keybindings.json` 을 열어 바꿀 수 있다 (저장하면 재시작 없이 적용). IntelliJ 터미널처럼 IDE가 단축키를 먼저 가로채면 IDE 쪽 설정도 확인한다.

🔗 [Keyboard shortcuts](https://code.claude.com/docs/en/interactive-mode#keyboard-shortcuts) · [Keybindings](https://code.claude.com/docs/en/keybindings)

---

## 핵심 정리

1. **출력 스타일 = 세션 전체의 "어떻게 답하나".** 지시일 뿐 보장은 아니다. 반드시 지킬 것은 훅으로 한다.
2. **내장 스타일은 Default 외 4가지**(Proactive, Concise, Explanatory, Learning). Proactive는 권한 모드를 바꾸지 않는다.
3. **`/output-style` 은 로컬 설정(`settings.local.json`)에 저장**된다. 설정 파일의 `outputStyle` 값은 대소문자를 구분한다.
4. **커스텀 스타일은 기본적으로 엔지니어링 지시를 뺀다.** 코딩을 계속 시킬 거면 `keep-coding-instructions: true`.
5. **스타일은 메인 대화와 fork에만 적용**되고, 일반 서브에이전트에는 적용되지 않는다.
6. **상태 표시줄 = stdin JSON → 스크립트 → stdout 한 줄(또는 여러 줄).** exit 0이어야 보이고, 워크스페이스 신뢰 전에는 실행되지 않는다.
7. **Windows는 Git Bash가 있으면 Git Bash로 실행**한다. 경로는 슬래시로, PowerShell 스크립트는 `powershell -NoProfile -File` 로 호출한다.
8. **느린 명령은 `session_id` 기반 캐시**로 5초마다만 계산한다.
9. **작업 중에 입력하면 중단이 아니라 대기열**에 들어간다. 곁가지 질문은 `/btw`, 긴 명령은 `Ctrl+B`.

## 실무에서 바로 써먹기

**상황**: Windows + IntelliJ에서 Java/JSP(Maven) 레거시를 유지보수한다. 신입 팀원은 코드베이스를 배우면서 작업하고, 숙련자는 짧은 응답을 원한다. 컨텍스트가 얼마나 찼는지 항상 보고 싶다. (경로·이름은 예시)

```text
[1. 팀 스타일 파일 공유]
  .claude/output-styles/korean-review.md 작성 (keep-coding-instructions: true)
  → 커밋해서 팀원 모두 /output-style 목록에서 고를 수 있게

[2. 사람마다 다른 스타일 선택]
  신입: /output-style explanatory   (Insight로 Spring/MyBatis 흐름을 익힘)
  숙련자: /output-style concise
  → 각자 settings.local.json 에 저장되므로 서로 충돌 없음

[3. 손으로 연습하고 싶을 때]
  /output-style learning
  → TODO(human) 자리에 직접 구현 → "다 했어" → Insight로 피드백

[4. 상태 표시줄 설정 (Git Bash 없는 PC)]
  C:\Users\me\.claude\statusline.ps1 작성 (모델, 폴더, ctx %)
  ~/.claude/settings.json → "command": "powershell -NoProfile -File C:/Users/me/.claude/statusline.ps1"
  모의 JSON을 파이프로 넣어 먼저 테스트

[5. 작업 중 컨텍스트 관리]
  ctx 가 50%를 넘으면 노란색 → 논리적 단위가 끝나면 /compact
  긴 mvn 빌드는 Ctrl+B 로 백그라운드
  "아까 그 매퍼 파일 이름 뭐였지?" → /btw (기록에 안 남음)

[6. 변경 검토]
  /diff 로 턴별 변경 확인 → IntelliJ diff 뷰어에서 최종 확인 → 커밋
```

## 참고한 공식 문서

- [Output styles](https://code.claude.com/docs/en/output-styles) · [Settings](https://code.claude.com/docs/en/settings) · [Prompt caching](https://code.claude.com/docs/en/prompt-caching)
- [Customize your status line](https://code.claude.com/docs/en/statusline)
- [Interactive mode](https://code.claude.com/docs/en/interactive-mode) · [Keybindings](https://code.claude.com/docs/en/keybindings) · [Commands](https://code.claude.com/docs/en/commands)
