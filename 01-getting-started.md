# 📗 1장. Claude Code 시작하기

> **기준**: 「클로드코드로 시작하는 실전 에이전틱 코딩」 CHAPTER 01 목차 + Claude Code 공식 문서([code.claude.com/docs](https://code.claude.com/docs), 2026-10-03 확인). 책과 달라진 부분은 **📌 현재 공식 문서 기준** 으로 표시했다.
>
> 관련 노트: [2장. 워크플로와 설정](./02-workflow-and-settings.md) · [3장. 에이전트 스킬](./03-agent-skills.md) · [4장. 서브에이전트](./04-subagents.md)

---

## 1-1. 클로드 코드와 에이전틱 코딩

### 엔트로픽과 에이전틱 AI의 부상

> 📌 공식 문서는 회사 역사나 AI 시장 흐름을 다루지 않는다. 여기서는 공식 문서에 있는 정의만 정리했다.

- **Claude Code란**: Anthropic(앤트로픽)이 만든 **에이전틱 코딩 도구**. "코드베이스를 읽고, 파일을 수정하고, 명령을 실행하고, 개발 도구와 연동한다."
- **에이전틱 하네스(agentic harness)**: 에이전트는 추론하는 **모델**과 행동하는 **도구**로 움직인다. Claude Code는 모델을 감싸서 도구를 제공하고, 모델이 보는 컨텍스트를 관리하는 계층이다.
- **모델**: 대부분의 코딩은 Sonnet, 복잡한 설계 판단은 Opus가 강하다. 세션 중에는 `/model`, 시작할 때는 `claude --model sonnet` 처럼 바꾼다.
- **쓸 수 있는 곳**: 터미널, VS Code, JetBrains(IntelliJ), 데스크톱 앱, 웹. 모두 같은 엔진이라 `CLAUDE.md`, 설정, MCP 서버를 함께 쓴다.

🔗 [Overview](https://code.claude.com/docs/en/overview) · [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)

### 에이전틱 코딩이란

**① 개념** — AI가 목표를 받으면 필요한 파일을 스스로 찾아 읽고, 수정하고, 명령을 실행해 검증까지 하는 방식.

**② 기존 AI 코딩과의 차이**

| 구분 | 인라인 코드 어시스턴트 (자동완성·채팅) | Claude Code (에이전틱) |
|---|---|---|
| 보는 범위 | 현재 파일 중심 | 프로젝트 전체 + 터미널 + git 상태 |
| 하는 일 | 코드 제안 | 관련 파일 검색 → 여러 파일 수정 → 테스트 실행 → (요청 시) 커밋 |
| 사람의 역할 | 제안을 복사·적용 | 목표와 맥락을 주고 위임, 중간에 방향 수정 |

**③ Claude Code의 작업 방식** — 공식 문서의 표현은 "**지시하지 말고 위임하라**". 어떤 파일을 읽고 어떤 명령을 실행할지는 Claude가 정한다. 또 대화형이라 완벽한 프롬프트가 필요 없다. 첫 시도가 틀리면 처음부터 다시 하지 말고 이어서 고쳐 나간다.

**④ 예시**

```text
주문을 취소해도 재고가 복구되지 않아.
관련 코드는 src/main/java/com/acme/order 쪽이야. 원인을 찾아서 고쳐줘.
```

→ Claude가 관련 파일을 검색하고, Service·DAO·매퍼를 읽고, 여러 파일을 함께 수정한 뒤 테스트로 확인한다.

🔗 [What Claude can access](https://code.claude.com/docs/en/how-claude-code-works#what-claude-can-access) · [Delegate, don't dictate](https://code.claude.com/docs/en/how-claude-code-works#delegate-dont-dictate)

### 에이전틱 루프: Claude Code의 동작 원리

**① 개념** — 작업을 받으면 **컨텍스트 수집 → 행동 → 결과 검증** 3단계를 반복한다. 세 단계는 딱 나뉘지 않고 섞여서 진행된다.

**② 왜 중요한가** — 각 도구 사용 결과가 다음 판단의 재료가 된다. 그래서 수십 개의 행동을 이어가며 스스로 방향을 고친다. 루프의 길이는 요청에 따라 다르다: 코드 질문은 수집만, 버그 수정은 3단계를 여러 번, 리팩터링은 검증을 많이 한다.

**③ 사용자도 루프의 일부다**

- `Esc`: 즉시 멈춘다. 실행 중인 도구 호출이 취소되고 다음 지시를 기다린다.
- 멈추지 않고 수정 지시를 입력하고 `Enter`: 메시지가 대기열에 들어가고, 진행 중인 도구 호출이 끝나면 바로 읽고 방향을 바꾼다.

**④ 예시 – "실패하는 테스트 고쳐줘"** (공식 예시를 Maven 프로젝트에 맞게 바꾼 것)

1. `mvn test` 를 실행해 실패한 테스트를 확인한다 (실행)
2. 에러 출력을 읽는다 (수집)
3. 관련 소스 파일을 검색한다 (수집)
4. 파일을 읽고 코드를 이해한다 (수집)
5. 파일을 수정한다 (행동)
6. 테스트를 다시 실행한다 (검증) → 실패하면 2번으로

💡 검증할 수단(테스트·빌드)이 있어야 루프가 스스로 닫힌다 → [2장](./02-workflow-and-settings.md) "점진적 개발 전략" 참고.

🔗 [The agentic loop](https://code.claude.com/docs/en/how-claude-code-works#the-agentic-loop) · [Interrupt and steer](https://code.claude.com/docs/en/how-claude-code-works#interrupt-and-steer)

### 터미널에서 시작하는 AI 코딩

**① 개념** — Claude Code는 터미널에서 실행되는 에이전트다. 내가 명령줄에서 할 수 있는 일(빌드 도구, git, 패키지 매니저, 스크립트)은 Claude도 할 수 있다.

**② 왜 터미널인가** — 이미 쓰고 있는 개발 도구를 그대로 활용하고, 파이프·스크립트와 조합할 수 있다 (상세는 2장 "파이프와 Claude Code").

**③ 사용할 수 있는 화면**

| 인터페이스 | 특징 |
|---|---|
| 터미널 CLI | 모든 기능을 쓸 수 있는 기본 형태 |
| VS Code 확장 | 그래픽 패널, 인라인 diff, 계획 검토. CLI를 내장하지만, 통합 터미널에서 `claude` 를 직접 쓰려면 CLI를 따로 설치해야 한다 |
| JetBrains 플러그인 (IntelliJ 등) | IDE diff 뷰어, 선택 영역 공유, IDE 진단(문법 오류 등) 읽기. **CLI를 따로 설치해야 한다** |
| 데스크톱 앱 / 웹 | 터미널 없이 사용, 여러 세션을 나란히 실행 |

**④ 예시 – IntelliJ에서 쓰기**

- IntelliJ **통합 터미널**에서 `claude` 를 실행하면 자동으로 연결된다. `Ctrl+Esc` 로 편집기에서 바로 열 수도 있다.
- 외부 터미널(Windows Terminal 등)에서는 `claude` 실행 후 `/ide` 로 연결한다. IDE와 같은 파일을 보려면 **프로젝트 루트에서 실행**한다.
- 코드를 선택한 채 질문하면 선택 영역이 자동으로 전달되고, 수정 내용은 IDE diff 뷰어로 떠서 바로 검토할 수 있다.

> ⚠️ **IntelliJ 주의점 (공식 문서)**
> - `Esc` 가 동작하지 않으면 Settings → Tools → Terminal에서 "Move focus to the editor with Escape"를 끈다.
> - acceptEdits·auto 모드에서는 IDE가 자동 실행하는 설정 파일까지 수정될 수 있다. 공식 문서는 편집을 Manual 모드로 승인하는 것을 고려하라고 권한다.

🔗 [JetBrains IDEs](https://code.claude.com/docs/en/jetbrains) · [VS Code](https://code.claude.com/docs/en/vs-code) · [Environments and interfaces](https://code.claude.com/docs/en/how-claude-code-works#environments-and-interfaces)

### 컨텍스트 윈도우와 20만 토큰 컨텍스트

**① 개념** — 한 세션에서 Claude가 동시에 볼 수 있는 작업 메모리. 대화 기록, 읽은 파일 내용, 명령 출력, `CLAUDE.md`, auto memory, 로드된 스킬, 시스템 지시가 모두 여기에 들어간다.

**② 왜 중요한가**

- 한도에 가까워지면 자동으로 정리한다: 오래된 도구 출력을 먼저 지우고, 필요하면 대화를 요약(compaction)한다. 이때 **대화 초반에 준 자세한 지시가 사라질 수 있다**.
- 공식 best practices: 컨텍스트가 차면 성능이 떨어지고, 앞의 지시를 "잊거나" 실수가 늘 수 있다. 그래서 컨텍스트는 **가장 중요하게 관리해야 하는 자원**이다.

**③ Claude Code에서 컨텍스트가 채워지는 순서**

- **아무것도 입력하기 전**: 시스템 프롬프트, `CLAUDE.md`, auto memory, MCP 도구 이름(상세 정의는 필요할 때 로드), 스킬 설명이 이미 들어 있다.
- **작업 중**: 파일을 읽을 때마다, 명령을 실행할 때마다 늘어난다.
- 현재 사용량은 `/context` 로 확인한다 (항목별 사용량 + 최적화 제안).

> 📌 **현재 공식 문서 기준 – "20만 토큰"은 모델에 따라 다르다**
> - Sonnet 5 이후, Opus 4.7 이후, Fable 모델은 **100만(1M) 토큰** 컨텍스트로 동작한다. Sonnet 5.5·Sonnet 5는 기본적으로 약 967K에서 자동 압축된다.
> - Opus 4.6·Sonnet 4.6은 기본 **20만(200K)** 이고, `[1m]` 변형(예: `opus[1m]`)을 고르면 1M을 쓸 수 있다 (플랜에 따라 추가 크레딧 필요).
> - 창이 커져도 **압축 방식과 관리 원칙은 같다**. 큰 창은 "관리하지 않아도 된다"는 뜻이 아니다.

**④ 대규모 프로젝트에서 컨텍스트가 부족해지는 상황**

- 범위 없이 "전체 조사해줘" → 수백 개 파일을 읽는다.
- 큰 로그·파일을 통째로 읽힌다 (예: 수만 줄짜리 `catalina.out`, 수천 줄짜리 레거시 JSP). 심하면 압축 직후 다시 가득 차서 `Autocompact is thrashing` 오류가 난다.
- 관련 없는 작업을 한 세션에 섞는다.
- 같은 문제를 여러 번 교정해서 실패한 시도가 쌓인다.

🔗 [The context window](https://code.claude.com/docs/en/how-claude-code-works#the-context-window) · [Explore the context window](https://code.claude.com/docs/en/context-window) · [Extended context](https://code.claude.com/docs/en/model-config#extended-context)

### 컨텍스트 윈도우 관리 전략

**① 불필요한 컨텍스트를 줄이는 명령**

| 방법 | 언제 |
|---|---|
| `/context` | 무엇이 공간을 차지하는지 확인할 때 |
| `/compact [지시]` | 긴 작업 중 공간을 확보할 때. 예: `/compact 주문 취소 버그 수정 내용만 유지` |
| `/clear` | 관련 없는 작업으로 넘어갈 때 (이전 대화는 `/resume` 으로 복귀 가능) |
| `/rewind` → 요약 | 대화 일부만 압축할 때 ("여기부터 요약" / "여기까지 요약") |
| `/autocompact 500k` | 자동 압축을 더 일찍 일어나게 할 때 |
| `/btw 질문` | 대화 기록에 남기지 않고 곁가지 질문을 할 때 (도구는 못 씀) |
| 서브에이전트 | 넓은 조사를 맡길 때: "use a subagent to investigate …" → 별도 컨텍스트에서 읽고 요약만 돌려받음 |

**② 압축 후에도 남는 것 / 사라지는 것**

- 남는 것: 프로젝트 루트 `CLAUDE.md`와 auto memory는 디스크에서 다시 읽는다. 최근 읽거나 수정한 파일은 최대 5개까지 다시 읽는다.
- 사라질 수 있는 것: 대화로만 전달한 지시. → **계속 지켜야 하는 규칙은 `CLAUDE.md` 에** 둔다.
- `CLAUDE.md`에 "Compact Instructions" 섹션을 두면 압축 때 무엇을 보존할지 지정할 수 있다.

**③ 프로젝트 지침 파일로 반복 설명 줄이기**

새 세션은 항상 빈 컨텍스트에서 시작한다. 매번 다시 설명하지 않으려면 아래 파일들을 쓴다. 모두 세션을 시작할 때 자동으로 로드된다.

| 파일 | 누가 / 범위 | 용도 |
|---|---|---|
| `./CLAUDE.md` 또는 `./.claude/CLAUDE.md` | 팀 공유 (git 커밋) | 빌드·테스트 명령, 코딩 규칙, 구조, 아키텍처 결정 |
| `./CLAUDE.local.md` | 나만, 이 프로젝트 (`.gitignore` 에 추가) | 내 로컬 URL, 테스트 데이터 |
| `~/.claude/CLAUDE.md` | 나만, 모든 프로젝트 | 개인 스타일, 선호 도구 |
| `.claude/rules/` | 주제별·경로별 규칙 | 예: JSP 규칙은 JSP 파일을 다룰 때만 로드 |
| auto memory (`MEMORY.md`) | Claude가 직접 기록 | 교정·선호에서 배운 점. 처음 200줄 또는 25KB 로드, `/memory` 로 확인·편집 |

**작성 원칙 (공식 문서)**

- `/init` 으로 초안을 만들고 다듬는다. 이미 있으면 덮어쓰지 않고 개선안을 제안한다.
- **파일당 200줄 이하**. 길수록 컨텍스트를 많이 쓰고 지시를 덜 따른다.
- 검증 가능하게 구체적으로: "코드 정리 잘해" 대신 "커밋 전에 `mvn test` 를 실행해".
- 언제 추가하나: Claude가 같은 실수를 두 번 했을 때, 지난 세션에 했던 설명을 또 하고 있을 때.
- `CLAUDE.md`는 **강제 설정이 아니라 컨텍스트**다. 반드시 막아야 하는 일은 권한 규칙·hook으로 막는다 (2장).

**예시 – Java/JSP 프로젝트 `CLAUDE.md`** (경로와 명령은 예시)

```markdown
# 빌드·테스트
- 빌드: mvn -q compile
- 테스트: mvn test -Dtest=클래스명 (전체 테스트보다 단일 테스트 우선)

# 구조
- Controller: src/main/java/com/acme/web
- JSP: src/main/webapp/WEB-INF/jsp
- SQL 매퍼: src/main/resources/mapper

# 규칙
- JSP에서는 스크립틀릿 대신 JSTL/EL을 쓴다
- 운영 설정 파일(application-prod*.properties)은 수정하지 않는다
```

🔗 [When your context fills up](https://code.claude.com/docs/en/context-window#when-your-context-fills-up) · [What survives compaction](https://code.claude.com/docs/en/context-window#what-survives-compaction) · [CLAUDE.md files](https://code.claude.com/docs/en/memory#claude-md-files) · [Auto memory](https://code.claude.com/docs/en/memory#auto-memory)

### Claude Code의 접근 영역과 핵심 기능

> 📌 **현재 공식 문서 기준**: 공식 문서의 "5가지" 분류는 **내장 도구의 범주**다. 책의 "다섯 가지 접근 영역"과 분류 방식이 다를 수 있어서, 아래에는 "접근 대상"과 "도구 범주"를 나눠 정리했다. 접근 대상에는 **auto memory**가 새로 포함됐다.

**① Claude가 접근하는 대상** — 프로젝트 폴더에서 `claude` 를 실행하면 다음에 접근한다.

| 대상 | 내용 |
|---|---|
| 프로젝트 | 현재 폴더와 하위 폴더의 파일. 바깥 파일은 허락을 받아야 한다 |
| 터미널 | 내가 실행할 수 있는 모든 명령 (빌드, git, 스크립트) |
| git 상태 | 현재 브랜치, 커밋하지 않은 변경, 최근 커밋 이력 |
| `CLAUDE.md` | 내가 쓴 프로젝트 지침 (`AGENTS.md`도 읽을 수 있음) |
| auto memory | Claude가 작업하며 저장한 학습 내용 |
| 확장 기능 | MCP 서버, 스킬, 서브에이전트, 브라우저 연동 |

**② 내장 도구 5가지 범주** — 도구가 있어야 Claude가 말만 하는 것이 아니라 실제로 행동한다.

| 범주 | 할 수 있는 일 | 실무에서 등장하는 곳 |
|---|---|---|
| 파일 작업 | 읽기, 수정, 생성, 이름 변경 | JSP·Java 파일 수정 |
| 검색 | 패턴으로 파일 찾기, 정규식 내용 검색 | "회원 탈퇴 처리 코드 찾아줘" |
| 실행 | 셸 명령, 서버 기동, 테스트, git | `mvn test`, `git diff` |
| 웹 | 웹 검색, 문서 가져오기, 에러 메시지 검색 | 라이브러리 문서 확인 |
| 코드 인텔리전스 | 수정 후 타입 오류 확인, 정의 이동, 참조 찾기 (**플러그인 필요**) | 큰 Java 프로젝트의 참조 추적 |

이 밖에 서브에이전트 실행, 사용자에게 질문하기 같은 조율용 도구가 있다. 기능 확장은 스킬·MCP·hooks·서브에이전트로 한다 ([3장](./03-agent-skills.md), [4장](./04-subagents.md)).

**③ 안전장치 2가지** — **체크포인트**(파일 변경 되돌리기)와 **권한**(묻지 않고 할 수 있는 일 제한). DB·API·배포처럼 원격 시스템에 영향을 주는 작업은 체크포인트로 되돌릴 수 없으므로 권한으로 통제한다.

🔗 [Tools](https://code.claude.com/docs/en/how-claude-code-works#tools) · [What Claude can access](https://code.claude.com/docs/en/how-claude-code-works#what-claude-can-access) · [Stay safe with checkpoints and permissions](https://code.claude.com/docs/en/how-claude-code-works#stay-safe-with-checkpoints-and-permissions)

### 엔터프라이즈 환경에서의 Claude Code

**① 개념** — 회사에서는 "어떤 계정·결제로 쓸지", "사내 네트워크를 어떻게 통과할지", "보안 정책을 어떻게 강제할지"를 정해야 한다.

**② 배포 방식**

| 방식 | 특징 |
|---|---|
| Claude for Teams / Enterprise | 대부분의 조직에 권장. SSO, 관리 도구, 서버 관리 설정. Enterprise는 도메인 캡처, 역할 기반 권한, 컴플라이언스 API 추가 |
| Claude Console | 개인 개발자, 사용량 기반 과금 |
| Amazon Bedrock / Claude Platform on AWS | AWS 기반 (IAM, CloudTrail) |
| Google Cloud's Agent Platform | GCP 기반. 📌 예전 이름은 Vertex AI |
| Microsoft Foundry | Azure 기반 (RBAC, Azure Monitor) |

**③ 사내 네트워크와 정책**

- 사내 프록시: `HTTPS_PROXY` / `HTTP_PROXY`. 중앙에서 인증·사용량을 관리하는 LLM 게이트웨이: `ANTHROPIC_BASE_URL` 등. 지금 어떤 provider·URL·프록시를 쓰는지는 `/status` 로 확인한다.
- 보안팀은 managed settings로 사용자가 덮어쓸 수 없는 권한 정책을 배포할 수 있다 (상세는 2장).

**④ 공식 문서의 조직 도입 권장사항**

- `CLAUDE.md` 등 문서화에 투자한다.
- 설치를 "원클릭"으로 단순하게 만든다.
- 처음에는 **코드베이스 Q&A나 작은 버그 수정**부터 시작하고, 계획을 먼저 세우게 해서 확인한다.
- 클라우드 provider를 쓰면 모델 버전을 고정한다 (`ANTHROPIC_DEFAULT_OPUS_MODEL` 등).
- MCP는 중앙 팀이 설정해 `.mcp.json` 을 저장소에 커밋한다.

🔗 [Enterprise deployment overview](https://code.claude.com/docs/en/third-party-integrations) · [Network configuration](https://code.claude.com/docs/en/network-config)

---

## 1-2. 클로드 코드 설치

### 운영체제별 설치 방법

**① 개념** — 공식 권장은 **native installer**다. 백그라운드에서 자동으로 업데이트된다.

**② 설치 명령**

```powershell
# Windows PowerShell (권장)
irm https://claude.ai/install.ps1 | iex
```

```bat
REM Windows CMD
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

```bash
# macOS, Linux, WSL
curl -fsSL https://claude.ai/install.sh | bash
```

```powershell
# 패키지 매니저 (자동 업데이트 없음)
winget install Anthropic.ClaudeCode          # Windows
brew install --cask claude-code               # macOS (stable 채널)
npm install -g @anthropic-ai/claude-code      # Node.js 22 이상, sudo 사용 금지
```

```powershell
# 설치 확인 (새 터미널에서)
claude --version
```

> 📌 **현재 공식 문서 기준**: npm이 아니라 **native installer가 권장 방식**이다. npm으로 설치해도 실제로는 같은 native 바이너리가 설치되며 Node.js 없이 실행된다. Windows는 **WSL 없이 네이티브로 설치**할 수 있고 관리자 권한도 필요 없다.

**③ Windows: 네이티브 vs WSL**

| 방식 | 필요한 것 | 샌드박스 | 언제 |
|---|---|---|---|
| 네이티브 Windows | 없음 (Git for Windows는 선택) | 미지원 | Windows 프로젝트·도구 (IntelliJ + Maven 등) |
| WSL 2 | WSL 2 | 지원 | Linux 툴체인, 샌드박스 필요 시 |
| WSL 1 | WSL 1 | 미지원 | WSL 2를 쓸 수 없을 때 |

- **Git for Windows가 있으면** Bash 도구(Git Bash)와 PowerShell 도구를 함께 쓴다. **없으면** PowerShell로 명령을 실행한다. Git Bash를 못 찾으면 settings의 `env` 에 `CLAUDE_CODE_GIT_BASH_PATH` 를 지정한다.
- 시스템 요구사항: Windows 10 1809 이상 또는 Windows Server 2019 이상, RAM 4GB 이상, 인터넷 연결.
- WSL에서 `/mnt/c/` 아래 프로젝트를 열면 파일시스템 성능 때문에 검색 결과가 적게 나올 수 있다 → Linux 파일시스템(`/home/`)으로 옮기거나 네이티브 Windows를 쓴다.
- IDE: VS Code는 Extensions에서 "Claude Code" 설치, IntelliJ는 JetBrains Marketplace에서 플러그인 설치 후 IDE 재시작 (CLI 별도 설치 필요).

**④ 자주 겪는 설치 오류 (Windows)**

- `The token '&&' is not a valid statement separator` → PowerShell에서 CMD용 명령을 실행한 것이다.
- `'irm' is not recognized …` → CMD에서 PowerShell용 명령을 실행한 것이다. (프롬프트가 `PS C:\` 면 PowerShell)
- `claude` 를 찾을 수 없음 → 새 터미널을 열고, 그래도 안 되면 PATH 문제다 (troubleshoot-install 문서).

🔗 [Advanced setup](https://code.claude.com/docs/en/setup) · [Set up on Windows](https://code.claude.com/docs/en/setup#set-up-on-windows) · [Troubleshoot installation](https://code.claude.com/docs/en/troubleshoot-install)

### 인증 설정

**① 개념** — Claude Code를 쓰려면 계정이 필요하다: Claude Pro/Max/Team/Enterprise, Claude Console, 또는 클라우드 provider(Bedrock 등). **무료 `claude.ai` 플랜에는 Claude Code가 포함되지 않는다.**

**② 로그인 흐름**

1. 처음 `claude` 를 실행하면 브라우저가 열리고 로그인한다.
2. 브라우저가 안 열리면 `c` 를 눌러 로그인 URL을 복사해 직접 연다.
3. 브라우저에 코드가 표시되면(WSL2, SSH 등) 터미널의 `Paste code here if prompted` 에 붙여넣는다.
4. `Login successful` 이 나오면 완료. 이후에는 다시 로그인할 필요가 없다.

**③ 관련 명령**

| 명령 | 설명 |
|---|---|
| `/login` | 세션 안에서 계정 전환·재인증 |
| `/logout` | 로그아웃 (다음 실행 때 로그인부터 다시 안내) |
| `claude auth status` | 셸에서 인증 상태를 JSON으로 확인 (`--text` 는 읽기 쉬운 형식) |
| `/status` | 현재 쓰는 인증 방식·계정 확인 |
| `claude setup-token` | CI·스크립트용 1년짜리 토큰 발급 → `CLAUDE_CODE_OAUTH_TOKEN` 으로 설정 (구독 플랜 필요) |

> ⚠️ **`ANTHROPIC_API_KEY` 주의** — 이 환경 변수가 있으면 구독 대신 **그 API 키로 과금**된다. 대화형에서는 처음에 한 번 승인을 묻고, `-p` 에서는 묻지 않고 항상 사용한다. 구독으로 쓰려면 변수를 제거하고 `/status` 로 확인한다.

**인증 우선순위** (여러 개가 있을 때): 클라우드 provider 변수 → `ANTHROPIC_AUTH_TOKEN` → `ANTHROPIC_API_KEY` → `apiKeyHelper` → `CLAUDE_CODE_OAUTH_TOKEN` → Anthropic profile → `/login` 구독 로그인

**알아두면 좋은 것**

- 인증 정보 저장 위치 (Windows): `%USERPROFILE%\.claude\.credentials.json` (내 사용자 계정만 접근 가능).
- 회사·개인 계정을 동시에 쓰려면 `CLAUDE_CONFIG_DIR` 로 설정 폴더를 나눈다 (폴더별로 로그인·설정·세션이 따로 관리됨).
- 📌 Claude Console 계정은 이제 **API 키를 만들지 않고도 로그인**할 수 있다 (v2.1.242 이상).
- 로그인 만료 3일 전부터 시작 시 경고가 뜨며, `/login` 으로 갱신한다.

🔗 [Authentication](https://code.claude.com/docs/en/authentication) · [Authentication precedence](https://code.claude.com/docs/en/authentication#authentication-precedence)

### 버전 관리와 업데이트 전략

**① 개념** — Claude Code는 자주 업데이트된다. 설치 방식에 따라 업데이트 방법이 다르다.

| 설치 방식 | 업데이트 |
|---|---|
| native installer | **자동**: 시작 시와 실행 중 확인 → 백그라운드에서 설치 → 다음 실행 때 적용 |
| WinGet | 수동: `winget upgrade Anthropic.ClaudeCode` |
| Homebrew | 수동: `brew upgrade claude-code` |
| npm | `npm install -g @anthropic-ai/claude-code@latest` (`npm update -g` 는 피한다) |

WinGet·Homebrew도 자동으로 하려면 `CLAUDE_CODE_PACKAGE_MANAGER_AUTO_UPDATE=1` 을 설정한다. (WinGet은 Claude Code 실행 중에는 파일 잠금 때문에 실패할 수 있다.)

**② 버전 관리 명령**

```powershell
claude --version     # 현재 버전 확인
claude update        # 지금 바로 업데이트
claude doctor        # 마지막 자동 업데이트 결과·설치 상태 확인

# 특정 버전 또는 stable 채널로 설치 (PowerShell)
& ([scriptblock]::Create((irm https://claude.ai/install.ps1))) stable
& ([scriptblock]::Create((irm https://claude.ai/install.ps1))) 2.1.89
```

**③ 릴리스 채널과 버전 고정**

- `latest`(기본): 새 기능을 바로 받는다.
- `stable`: 보통 약 1주 전 버전이며, 큰 회귀가 있던 릴리스는 건너뛴다. 단 새 모델은 최신 버전이 필요할 수 있다.
- 설정: `/config` → Auto-update channel, 또는 settings에 `"autoUpdatesChannel": "stable"`.
- `minimumVersion`: 이 버전 아래로는 업데이트되지 않게 하한을 건다.

**④ 업데이트 끄기**

| 변수 | 효과 |
|---|---|
| `DISABLE_AUTOUPDATER=1` | 백그라운드 자동 업데이트만 끈다 (`claude update` 는 동작) |
| `DISABLE_UPDATES` | 수동 업데이트까지 모든 경로를 막는다 (회사가 버전을 직접 배포할 때) |

💡 **실무 제안 (공식 권장안 아님)**: 회사 PC처럼 안정성이 중요하면 `stable` 채널을 고려한다. 조직 단위로는 managed settings로 채널·최소 버전을 강제할 수 있다.

🔗 [Update Claude Code](https://code.claude.com/docs/en/setup#update-claude-code) · [Configure release channel](https://code.claude.com/docs/en/setup#configure-release-channel) · [Install a specific version](https://code.claude.com/docs/en/setup#install-a-specific-version)

---

## 1-3. 클로드 코드 CLI 기초

### 대화형 모드와 비대화형 모드

**① 개념 비교**

| 방식 | 특징 | 사용 상황 |
|---|---|---|
| 대화형 `claude` | Claude와 대화하며 작업. 권한 확인, 중간 방향 수정, 되감기 가능 | 코드 분석, 개발, 디버깅 |
| 비대화형 `claude -p` | 한 번 실행하고 결과를 출력한 뒤 종료. 파이프 입력·JSON 출력 가능 | 자동화, CI/CD, 스크립트, 일괄 처리 |

**② 기본 명령**

```bash
claude
```

현재 폴더에서 대화형 세션을 시작한다.

```bash
claude "빌드 에러 고쳐줘"
```

첫 프롬프트를 넣은 채로 대화형 세션을 시작한다.

```bash
claude -p "프로젝트 구조를 분석하고 주요 모듈을 설명해줘"
```

결과만 출력하고 종료한다 (비대화형).

```bash
claude -c
```

이 폴더의 가장 최근 대화를 이어간다.

```bash
claude -r
```

이전 대화 목록에서 골라 이어간다.

**③ 대화형 모드에서 꼭 알아야 할 입력**

| 입력 | 동작 |
|---|---|
| `/` | 명령·스킬 목록 (`/help` 로 전체 도움말) |
| `@파일경로` | 파일을 바로 대화에 포함 |
| `!` 뒤에 명령 | 셸 명령을 직접 실행하고 그 출력을 대화에 넣음 (예: `! git status`) |
| `Esc` | Claude 작업 중단 |
| `Esc` 두 번 | 되감기 메뉴 (입력창이 비어 있을 때) |
| `Shift+Tab` | 권한 모드 전환 (auto → Manual → acceptEdits → plan) |
| `Ctrl+C` | 작업 중단 / 입력 지우기, 아무것도 없을 때 두 번 누르면 종료 |
| `/exit` 또는 `Ctrl+D` 두 번 | 종료 |
| `Shift+Enter` | 줄바꿈 (Windows Terminal은 기본 지원, VS Code 터미널 등은 `/terminal-setup`) |
| `Alt+V` | Windows에서 클립보드 이미지 붙여넣기 (에러 화면 캡처 등) |
| `↑` / `Ctrl+R` | 입력 기록 / 기록 검색 |
| `Ctrl+G` | 긴 프롬프트를 외부 편집기에서 작성 |

비대화형 모드의 출력 형식, 세션 이어가기, 파이프 활용은 [2장](./02-workflow-and-settings.md)에 정리했다.

🔗 [Quickstart – Essential commands](https://code.claude.com/docs/en/quickstart#essential-commands) · [Interactive mode](https://code.claude.com/docs/en/interactive-mode) · [CLI reference](https://code.claude.com/docs/en/cli-reference)

### 주요 실행 플래그

**① 개념** — `claude` 를 실행할 때 붙이는 옵션. 그 세션의 모델, 권한, 작업 폴더 등을 정한다. 처음 배울 때 쓸 것만 골랐다.

| 플래그 | 하는 일 | 예 |
|---|---|---|
| `--model` | 이 세션의 모델 지정 | `claude --model opus` |
| `--permission-mode` | 지정한 권한 모드로 시작 (`default`, `acceptEdits`, `plan`, `auto`, `dontAsk`) | `claude --permission-mode plan` |
| `-n`, `--name` | 세션 이름 지정 (나중에 `claude -r 이름`) | `claude -n login-lock` |
| `-c`, `-r` | 최근 대화 이어가기 / 골라서 이어가기 | `claude -r login-lock` |
| `--add-dir` | 다른 폴더도 읽고 쓸 수 있게 추가 | `claude --add-dir ..\common-lib` |
| `-p` | 비대화형 실행 | `claude -p "요약해줘"` |
| `-v`, `--version` | 버전 출력 | `claude -v` |
| `--verbose` | 턴별 상세 출력 | `claude --verbose` |
| `--debug` | 디버그 모드 (문제 분석용) | `claude --debug` |
| `--safe-mode` | `CLAUDE.md`·플러그인·hooks·MCP 등 모든 커스터마이징을 끄고 실행 (원인 격리) | `claude --safe-mode` |
| `--ide` | 시작할 때 IDE에 자동 연결 | `claude --ide` |

> 📌 **현재 공식 문서 기준**: `--enable-auto-mode` 는 제거됐다 → `--permission-mode auto` 를 쓴다. `claude --help` 에 모든 플래그가 나오는 것은 아니므로 전체 목록은 CLI reference에서 확인한다.

🔗 [CLI flags](https://code.claude.com/docs/en/cli-reference#cli-flags)

### 시스템 상태 진단과 문제 해결

**① 개념** — 설치·설정·연결 상태를 확인하는 명령들. "어디서 실행하느냐(셸 / 세션 안)"를 구분하면 쉽다.

| 명령 | 실행 위치 | 하는 일 |
|---|---|---|
| `claude --version` | 셸 | 설치 확인 |
| `claude doctor` | 셸 | **읽기 전용** 진단: 설치 상태, 설정 파일 오류, 자동 업데이트 결과. `claude` 가 시작조차 안 될 때 |
| `/doctor` | 세션 안 | 설치·설정·확장·컨텍스트 사용량 점검 + **수정 제안** (확인 후 적용) |
| `/status` | 세션 안 | 버전, 모델, 계정, 연결 상태, 로드된 설정 파일 |
| `/context` | 세션 안 | 컨텍스트 사용량 |
| `/usage` | 세션 안 | 세션 비용, 플랜 사용 한도 (📌 `/cost` 는 이제 별칭) |
| `/mcp` | 세션 안 | MCP 서버 상태 |
| `/feedback` | 세션 안 | Anthropic에 문제 보고 |

**② 상황별 해결법**

| 증상 | 해결 |
|---|---|
| 응답이 멈춤 | `Ctrl+C` → 안 되면 터미널 재시작 후 `claude --resume` (대화는 유실되지 않음) |
| 느려짐·메모리 과다 | `/compact`, 큰 작업 사이에 재시작, 큰 빌드 폴더(예: `target/`)를 `.gitignore` 에 추가 |
| `Autocompact is thrashing` 오류 | 큰 파일을 범위를 나눠 읽게 하기, `/compact` 에 유지할 내용 지정, 서브에이전트에 맡기기, `/clear` |
| 검색·`@` 참조가 파일을 못 찾음 | 내장 ripgrep 문제 가능 → `winget install BurntSushi.ripgrep.MSVC` 후 `USE_BUILTIN_RIPGREP=0` |
| 플러그인·hook 때문인지 확인 | `claude --safe-mode` 로 실행해 비교 |
| 터미널 글자가 깨짐 (VS Code 터미널) | `/terminal-setup` |
| 설정이 적용되지 않음 | `/status` 로 로드된 파일 확인, `claude doctor` (2장 참고) |

🔗 [Troubleshooting](https://code.claude.com/docs/en/troubleshooting) · [Commands](https://code.claude.com/docs/en/commands) · [Debug your configuration](https://code.claude.com/docs/en/debug-your-config)

### 체크포인트와 되감기

**① 개념** — Claude가 **파일 편집 도구로 파일을 고치기 전에** 현재 내용을 자동으로 저장해 둔다. 프롬프트를 보낼 때마다 체크포인트가 하나씩 생긴다.

**② 왜 필요한가** — 다른 구현 방식을 시도해 볼 때, 버그를 만든 변경을 빨리 되돌릴 때, 긴 디버깅 대화를 요약해 컨텍스트를 비울 때.

**③ 사용법** — 입력창이 비어 있을 때 `Esc` 두 번, 또는 `/rewind` (별칭 `/checkpoint`, `/undo`). "방금 변경 되돌려줘"라고 요청해도 된다.

| 메뉴 | 결과 |
|---|---|
| Restore code and conversation | 코드와 대화를 모두 그 시점으로 |
| Restore conversation | 대화만 되돌리고 코드는 유지 |
| Restore code | 코드만 되돌리고 대화는 유지 |
| Summarize from here / up to here | 그 지점 이후 / 이전 대화를 요약 (파일은 그대로) |

VS Code 확장에서는 메시지에 마우스를 올리면 나오는 rewind 버튼으로 같은 기능을 쓴다.

**④ 상황별 사용**

- **코드 수정 전후** — 수정 전 상태는 자동으로 저장되므로 따로 백업할 필요가 없다. 수정 후에는 `/diff` 나 IntelliJ diff 뷰어로 변경 내용을 확인한다.
- **잘못된 변경** — 문제가 생긴 프롬프트 지점을 골라 **Restore code** 를 고르면, 대화는 남아 있어서 "왜 틀렸는지" 이어서 설명할 수 있다. 완전히 다시 하려면 코드와 대화를 모두 되돌리고 프롬프트를 고쳐 보낸다.
- **이전 상태로 복구** — 세션당 최근 100개 체크포인트를 보관하고, 세션을 재개한 뒤에도 `/rewind` 를 쓸 수 있다. 스냅샷은 기본 약 30일 뒤 정리된다 (`cleanupPeriodDays`). `/clear` 하기 전 대화도 같은 프로세스 안이면 되감기 메뉴 맨 위 항목으로 돌아갈 수 있다.
- **Git과의 관계** — 아래 참고.

**되돌릴 수 없는 것 (한계)**

- Bash·PowerShell 명령으로 바뀐 파일 (`rm`, `mv`, `cp`, 빌드 산출물 등)
- Claude Code 밖에서 직접 고친 파일, 다른 세션이 고친 파일 (대부분 추적 안 됨)
- 대부분의 서브에이전트 편집 → git으로 되돌린다
- 심볼릭 링크·하드 링크 파일
- DB, API, 배포 같은 원격 작업

> ⚖️ **Git `reset` / `revert` 와 같은 개념인가? → 아니다.**
>
> 공식 문서: 체크포인트는 **git과 별개**이고("separate from git"), **버전 관리를 대체하지 않는다**("not a replacement for version control"). 세션 안에서 빠르게 복구하는 용도이고, 영구 이력·협업은 git을 쓴다.
>
> 공식 문서가 `reset`/`revert` 를 직접 비교하지는 않는다. 아래 표는 공식 설명을 바탕으로 정리한 것이다.

| 구분 | Claude Code 되감기 | git reset / revert |
|---|---|---|
| 대상 | 이 세션에서 Claude의 파일 편집 도구가 바꾼 파일 + 대화 | 저장소의 커밋 이력과 작업 트리 |
| 기록 위치·기간 | 내 PC의 세션 기록, 기본 약 30일 | 저장소에 영구 기록, 팀과 공유 |
| 용도 | 세션 안 빠른 실험·복구 | 영구 버전 관리·협업 |
| Claude가 실행한 `git commit` | 되돌리지 못함 (셸 명령은 추적 대상이 아님) | reset / revert 로 처리 |

💡 **실무 원칙**: 작업 전에 git 상태를 깨끗하게(커밋 또는 stash) 해두고 → 세션 안 실험은 되감기로 → 확정되면 커밋한다.

🔗 [Checkpointing](https://code.claude.com/docs/en/checkpointing) · [Undo changes with checkpoints](https://code.claude.com/docs/en/how-claude-code-works#undo-changes-with-checkpoints)

---

## 1-4. 프로젝트로 특징 이해하기

### 프로젝트 컨텍스트 이해

**① 개념** — Claude는 필요한 파일을 스스로 읽는다. 컨텍스트를 일일이 넣어줄 필요가 없다.

**② 왜 먼저 하는가** — 구조를 이해시킨 뒤에 수정을 맡기면 엉뚱한 곳을 고치는 일이 줄어든다. 넓은 질문에서 좁은 질문 순서로 진행한다.

**③ 순서**

```powershell
cd C:\work\acme-shop     # 프로젝트 루트에서
claude
```

```text
이 프로젝트가 무엇을 하는지 설명해줘
어떤 기술 스택을 쓰고 있어? (프레임워크, 서블릿 버전, DB, 빌드 도구)
진입점과 폴더 구조를 설명해줘
로그인 요청이 JSP에서 DB까지 어떻게 흐르는지 추적해줘
이 프로젝트에서만 쓰는 용어를 정리해줘
```

**④ 파악한 내용을 남기기** — `/init` 을 실행하면 Claude가 코드베이스를 분석해 `CLAUDE.md` 초안(빌드 명령, 테스트 방법, 규칙)을 만든다. 검토하고 다듬은 뒤 커밋하면 팀 모두가 다음 세션부터 같은 맥락으로 시작한다. 로드 여부는 `/context` 의 Memory files에서 확인한다.

🔗 [Quickstart – Ask your first question](https://code.claude.com/docs/en/quickstart#step-4-ask-your-first-question) · [Understand new codebases](https://code.claude.com/docs/en/common-workflows#understand-new-codebases) · [Set up a project CLAUDE.md](https://code.claude.com/docs/en/memory#set-up-a-project-claude-md)

### 간단한 프로그램 작성

**① 개념** — 원하는 것을 자연어로 설명하면 Claude가 알맞은 파일을 찾아 변경을 만든다.

**② 왜 작은 것부터** — 공식 문서도 처음에는 작은 버그 수정·기능 추가로 시작해 Claude의 작업 방식에 익숙해지라고 권한다.

**③ 예시**

```text
WEB-INF/jsp/member/join.jsp 회원가입 폼에 이메일 형식 검증을 추가해줘.
서버 쪽 MemberController에도 같은 검증을 넣고,
MemberControllerTest에 테스트를 추가한 뒤 mvn test로 확인해줘.
```

진행: 관련 파일 탐색 → 변경 제시 → (권한 모드에 따라) 승인 → 테스트 실행 → 결과 보고

> 📌 **현재 공식 문서 기준**: v2.1.283부터 대화형 세션은 **auto 모드**로 시작한다. 분류기 모델이 위험한 행동만 막고 대부분은 묻지 않고 진행한다. 처음 배울 때나 회사 소스라면 `Shift+Tab` 으로 **Manual** 모드로 바꿔 변경을 하나씩 보는 것도 방법이다 (권한 상세는 2장).

🔗 [Quickstart – Make your first code change](https://code.claude.com/docs/en/quickstart#step-5-make-your-first-code-change)

### 파일 읽기와 쓰기

**① 개념** — 읽기는 작업 폴더 안이면 승인 없이 필요할 때 한다. 쓰기는 파일 편집 도구로 하며, 수정 전 상태가 체크포인트에 남는다.

**② 왜 패턴을 알아야 하나** — 파일을 어떻게 가리키느냐에 따라 정확도와 컨텍스트 사용량이 달라진다.

**③ 패턴**

| 패턴 | 예 | 언제 |
|---|---|---|
| Claude가 찾게 하기 | "회원 탈퇴를 처리하는 파일들을 찾아줘" | 위치를 모를 때 |
| `@파일` | `@src/main/java/com/acme/member/MemberDAO.java 설명해줘` | 위치를 알 때 (내용 즉시 포함) |
| `@디렉터리` | `@src/main/webapp/WEB-INF/jsp 구조 알려줘` | 폴더 구성 파악 (목록만, 내용은 없음) |
| IntelliJ 선택 영역 | 코드를 선택하고 질문 / `Alt+Ctrl+K` 로 파일 참조 삽입 | IDE에서 코드를 보면서 |
| 이미지 | `Alt+V` 로 에러 화면 붙여넣기 | 화면 오류, UI 시안 |
| 다른 폴더 | `--add-dir`, `/add-dir` | 공통 모듈이 다른 폴더에 있을 때 |
| 수정 검토 | `/diff`, IntelliJ diff 뷰어 | 쓰기 직후 |

**④ 알아둘 점**

- `@파일` 로 참조하면 그 폴더와 상위 폴더의 `CLAUDE.md`도 함께 들어온다. 하위 폴더의 `CLAUDE.md`는 그 폴더 파일을 읽을 때 로드된다.
- 큰 파일은 "OrderService의 cancel 메서드만 봐줘"처럼 범위를 지정해서 읽게 한다 (컨텍스트 절약).
- 민감한 파일(.env, 운영 설정)을 읽지 못하게 막는 방법은 2장 "민감한 파일 보호" 참고.

🔗 [Reference files and directories](https://code.claude.com/docs/en/common-workflows#reference-files-and-directories) · [Review changes with /diff](https://code.claude.com/docs/en/interactive-mode#review-changes-with-%2Fdiff)

### Git 통합

**① 개념** — Claude는 현재 브랜치, 커밋하지 않은 변경, 최근 커밋 이력을 알고 시작한다. Git 작업을 대화로 요청할 수 있다.

**② 왜 유용한가** — 변경 요약, 커밋 메시지 작성, 충돌 해결 같은 반복 작업을 줄여준다.

**③ 예시 프롬프트**

```text
내가 바꾼 파일이 뭐야?
변경 내용을 설명하는 메시지로 커밋해줘
feature/login-lock 브랜치를 만들어줘
최근 5개 커밋을 보여줘
머지 충돌 해결을 도와줘
```

**④ 알아둘 점**

- 브랜치를 바꾸면 Claude는 새 브랜치의 파일을 보지만 **대화 기록은 그대로** 유지된다.
- 커밋에 `Co-Authored-By` 같은 문구가 붙는 것은 Claude Code가 추가하는 지침 때문이다 → `attribution` 설정으로 바꾼다 (2장).
- 커밋 전에는 `/diff` 나 `git diff` 로 변경을 직접 확인하자.
- PR 생성, worktree 병렬 작업, push 확인 규칙은 [2장](./02-workflow-and-settings.md) "Git 통합 워크플로".

🔗 [Quickstart – Use Git with Claude Code](https://code.claude.com/docs/en/quickstart#step-6-use-git-with-claude-code) · [Work across branches](https://code.claude.com/docs/en/how-claude-code-works#work-across-branches)

### 효과적인 작업 관리

**① 구체적으로 요청한다**

- ❌ "로그인 버그 고쳐줘"
- ✅ "비밀번호를 틀리면 빈 화면이 나오는 로그인 버그를 고쳐줘. LoginController와 login.jsp 쪽을 봐줘"

**② 단계로 나눈다**

```text
1. 회원 프로필 테이블을 만드는 SQL을 작성해줘
2. 프로필 조회·수정 DAO와 Service를 만들어줘
3. 프로필 수정 JSP 화면을 만들어줘
```

**③ 먼저 탐색시킨다** — 수정 전에 "관련 구조를 먼저 분석해줘". 큰 작업은 Plan 모드로 계획부터 세운다 (2장 "탐색 → 계획 → 구현 → 커밋").

**④ 위임하되 맥락을 준다** — 증상, 예상 위치, "고쳐진 상태"가 무엇인지를 말해 준다.

**⑤ 중간에 바로잡는다** — 빗나가면 `Esc` 로 멈추거나, 멈추지 않고 수정 지시를 입력한다.

**⑥ 작업 하나 = 세션 하나** — 관련 없는 일로 넘어갈 때 `/clear`. 세션에 이름을 붙이고(`claude -n 이름`, `/rename`) 나중에 `claude -r 이름` 으로 돌아온다.

**⑦ Claude에게 물어본다** — Claude는 자기 문서를 알고 있다. "hooks는 어떻게 설정해?"처럼 물어보면 된다.

🔗 [Quickstart – Pro tips for beginners](https://code.claude.com/docs/en/quickstart#pro-tips-for-beginners) · [Work effectively with Claude Code](https://code.claude.com/docs/en/how-claude-code-works#work-effectively-with-claude-code)

### 마무리

1장에서는 Claude Code가 **모델 + 도구 + 컨텍스트 관리**로 이뤄진 에이전트이고, 에이전틱 루프로 작업한다는 것을 봤다. 설치·인증·업데이트, 대화형/비대화형 사용법, 진단 도구, 되감기도 익혔다. 다음 단계인 계획 중심 워크플로, 설정·권한 관리는 [2장. 워크플로와 설정](./02-workflow-and-settings.md)에서 이어진다.

---

## 핵심 정리

1. **Claude Code는 자동완성 도구가 아니라 에이전트다.** 프로젝트 전체를 탐색하고, 여러 파일을 수정하고, 명령을 실행해 검증한다.
2. **에이전틱 루프 = 컨텍스트 수집 → 행동 → 검증의 반복.** 사용자는 `Esc` 와 추가 메시지로 언제든 끼어들 수 있다.
3. **컨텍스트가 가장 중요한 자원이다.** `/context` 로 확인하고, `/compact`·`/clear`·서브에이전트로 관리한다. 현재 주력 모델은 1M이지만 관리 원칙은 같다.
4. **반복 설명은 `CLAUDE.md` 로.** `/init` 으로 시작하고 200줄 이하로 유지한다. 압축 후에도 다시 로드된다.
5. **설치는 native installer가 기본이다.** Windows는 네이티브로 설치할 수 있고, 샌드박스가 필요하면 WSL2를 쓴다.
6. **`ANTHROPIC_API_KEY` 가 있으면 구독보다 먼저 쓰인다.** 인증 방식은 `/status` 로 확인한다.
7. **대화형(`claude`)은 개발·분석용, 비대화형(`claude -p`)은 자동화용이다.**
8. **진단은 셸의 `claude doctor`(읽기 전용)와 세션의 `/doctor`(수정 제안)로 나뉜다.**
9. **되감기(`Esc` 두 번, `/rewind`)는 git이 아니다.** Claude의 파일 편집만 되돌리므로, 확정된 변경은 커밋으로 남긴다.
10. **작게, 구체적으로, 먼저 탐색시키고 위임한다.**

## 실무에서 바로 써먹기

**상황**: 회사 Java/JSP(Maven) 프로젝트에서 "로그인 5회 실패 시 계정 잠금"을 추가한다. Windows + IntelliJ + Git 환경. (경로·클래스명은 예시)

```text
[0. 준비]
  git status 로 작업 트리가 깨끗한지 확인 (필요하면 커밋·stash)
  git switch -c feature/login-lock

[1. 프로젝트 폴더에서 Claude Code 실행]
  IntelliJ 통합 터미널 (프로젝트 루트) → claude -n login-lock
  처음이면 /init 으로 CLAUDE.md 만들기

[2. 프로젝트 구조 탐색 요청]
  "로그인 요청이 login.jsp → LoginController → MemberService → MemberDAO로
   어떻게 흐르는지 추적해줘. 아직 수정하지 마."

[3. 관련 파일 분석]
  "로그인 실패 횟수를 저장할 곳이 있는지 테이블과 매퍼를 확인해줘"

[4. 작업 계획 확인]
  Shift+Tab 으로 plan 모드 → "계정 잠금을 추가하려면 어떤 파일을 바꿔야 해? 계획 세워줘"
  계획을 검토하고 수정할 점을 말한 뒤 승인

[5. 코드 수정]
  처음에는 Manual 모드로 변경을 하나씩 승인 (IntelliJ diff 뷰어에서 확인)
  빗나가면 Esc → 방향 수정, 잘못 고쳤으면 Esc 두 번 → Restore code

[6. 테스트 및 검증]
  "MemberServiceTest에 잠금 테스트를 추가하고 mvn test를 실행해서 실패하면 고쳐줘.
   테스트 출력도 보여줘."

[7. Git diff 확인]
  /diff 또는 ! git diff 로 전체 변경 검토
  "이 변경에서 위험한 부분이 있으면 짚어줘"

[8. 변경사항 커밋]
  "변경 내용을 설명하는 메시지로 커밋해줘"
  다음 작업은 /clear 후 시작 (이 작업은 나중에 claude -r login-lock)
```

## 참고한 공식 문서

- [Overview](https://code.claude.com/docs/en/overview) · [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works) · [Quickstart](https://code.claude.com/docs/en/quickstart)
- [Explore the context window](https://code.claude.com/docs/en/context-window) · [Model configuration](https://code.claude.com/docs/en/model-config) · [How Claude remembers your project](https://code.claude.com/docs/en/memory)
- [Enterprise deployment overview](https://code.claude.com/docs/en/third-party-integrations)
- [Advanced setup](https://code.claude.com/docs/en/setup) · [Authentication](https://code.claude.com/docs/en/authentication)
- [CLI reference](https://code.claude.com/docs/en/cli-reference) · [Commands](https://code.claude.com/docs/en/commands) · [Interactive mode](https://code.claude.com/docs/en/interactive-mode)
- [Troubleshooting](https://code.claude.com/docs/en/troubleshooting) · [Checkpointing](https://code.claude.com/docs/en/checkpointing)
- [Common workflows](https://code.claude.com/docs/en/common-workflows) · [JetBrains IDEs](https://code.claude.com/docs/en/jetbrains) · [VS Code](https://code.claude.com/docs/en/vs-code)
