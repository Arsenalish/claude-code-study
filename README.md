# Claude Code 스터디 노트

「클로드코드로 시작하는 실전 에이전틱 코딩: 완벽한 통제를 위한 AI 개발팀 구축 가이드」의 목차를 따라가며, 내용은 **Claude Code 공식 문서([code.claude.com/docs](https://code.claude.com/docs))를 기준으로 현재 버전에 맞게** 정리한 개인 학습 노트입니다.

| 장 | 내용 |
|---|---|
| [1장. Claude Code 시작하기](./01-getting-started.md) | 에이전틱 코딩, 에이전틱 루프, 컨텍스트 윈도우, 설치·인증·업데이트, CLI 기초, 체크포인트와 되감기 |
| [2장. 워크플로와 설정](./02-workflow-and-settings.md) | 탐색→계획→구현→커밋, Git 워크플로, 비대화형 모드·파이프, settings.json, 환경 변수, 민감 파일 보호, 권한 |
| [3장. 에이전트 스킬](./03-agent-skills.md) | SKILL.md 구조, 점진적 로딩, 컨텍스트 비용, 도구 권한, 커스텀 스킬, `context: fork` |
| [4장. 서브에이전트](./04-subagents.md) | 위임 패턴, 내장·커스텀 서브에이전트, frontmatter, 스킬 주입, 에이전트 팀, `--agent` |
| [5장. 출력 인터페이스](./05-output-interface.md) | 출력 스타일(내장·커스텀), 상태 표시줄(Bash·PowerShell·Python·Node.js), 대화형 모드 고급 제어와 단축키 |
| [6장. 메모리와 대화 세션 관리](./06-memory-and-sessions.md) | `CLAUDE.md` 계층, auto memory, import·조건부 로딩, `.claude/rules/`, 체크포인트 심화, 세션 재개·보관 전략 |
| [7장. 자동화](./07-automation.md) | `-p`·`--bare` 자동화, 훅(이벤트·종료 코드·매처·prompt/agent/HTTP 훅), MCP, 플러그인(LSP·마켓플레이스·배포) |

## 표시 규칙

- **📌 현재 공식 문서 기준** — 책 내용과 현재 Claude Code 동작이 달라진 부분
- **🔎 공식 문서 원칙에서 정리** — 공식 문서에 전용 설명이 없어 공식 원칙만으로 구성한 부분
- **(제안)** — 공식 문서가 아닌 실무 제안

## 참고

- 기준 시점: 2026년 10월 공식 문서. Claude Code는 업데이트가 잦으므로 최신 내용은 각 절의 🔗 공식 문서 링크에서 확인하세요.
- Java/JSP·Windows 예시의 경로, 클래스명, 규칙은 설명을 위한 가상의 예시입니다.
- 책의 본문을 옮긴 것이 아니라 목차만 참고했으며, 개인 학습 목적으로 작성했습니다.
