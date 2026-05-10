# agent-work-mem

[English README](README.md) | [日本語 README](README.ja.md) | [tmux 사용법](tmux-usage.ko.md)

> 여러 AI 코딩 에이전트를 하나의 팀처럼 쓰기 위한 파일 기반 공유 작업 기억입니다.
>
> Claude, Codex, Gemini, OpenCode, Cursor, Aider 같은 에이전트가 같은 프로젝트의 `AIMemory/`를 읽고, 작업을 이어받고, 서로 handoff할 수 있게 합니다.

---

## 왜 필요한가요?

좋은 AI 코딩 에이전트는 많습니다. 하지만 한 에이전트만 쓰기에는 토큰과 관점이 부족하고, 여러 에이전트를 쓰면 매번 맥락을 복사해서 붙여 넣어야 합니다.

`agent-work-mem`은 프로젝트 안에 `AIMemory/`라는 공유 작업 기억을 만듭니다. 에이전트는 새 세션을 시작할 때 이 기억을 읽고, 작업 중 생긴 결정과 결과를 markdown으로 남깁니다.

그래서 이런 흐름이 쉬워집니다.

```text
Codex가 설계한다 -> Claude가 리뷰한다 -> OpenCode가 구현한다
```

핵심은 간단합니다. 서버도, 데이터베이스도, 별도 SaaS도 없습니다. 프로젝트 폴더 안의 markdown 파일만 사용합니다.

---

## 설치

프로젝트 폴더에서 아무 에이전트나 열고 이렇게 말하세요.

```text
Fetch https://raw.githubusercontent.com/daystar7777/agent-work-mem/main/prompt.md and apply it to this project.
```

또는 더 편하게:

```text
Install daystar7777/agent-work-mem into this project.
```

에이전트가 `AIMemory/` 폴더를 만들고, 공유 작업 기억 프로토콜을 설치합니다.

설치가 끝난 뒤 다른 에이전트를 열면 이렇게 시작하면 됩니다.

```text
Read the project structure and AIMemory, then tell me you understand the current state.
```

에이전트는 `AIMemory/INDEX.md`, `AIMemory/PROJECT_OVERVIEW.md`, `AIMemory/work.log`를 읽고 현재 프로젝트 상태를 파악합니다.

---

## 기본 handoff 사용법

작업을 보내는 에이전트에게 자연어로 말하면 됩니다.

```text
로그인만 되는 웹사이트를 설계해줘. 설계가 끝나면 Claude에게 리뷰 handoff를 만들어줘.
```

받는 에이전트에게는 이렇게 말합니다.

```text
최근 handoff를 읽고 리뷰해줘.
```

그러면 에이전트가 `AIMemory/handoff_*.md` 파일을 만들고, `AIMemory/work.log`에 언제 누가 무엇을 넘겼는지 기록합니다.

handoff 파일을 사람이 직접 만들 필요는 없습니다. 평소처럼 말하면 에이전트가 구조화된 handoff 파일을 만들어 줍니다.

---

## tmux로 여러 에이전트를 동시에 쓰기

Codex, Claude, Gemini, OpenCode를 한 프로젝트 안에서 동시에 켜두고 싶다면 tmux가 가장 쉽습니다. 화면을 여러 pane으로 나누고, 각 pane에 에이전트를 하나씩 띄우면 됩니다.

더 자세한 한글판 절차는 [tmux 사용법](tmux-usage.ko.md)에 따로 정리해 두었습니다.

### 1. 프로젝트 폴더에서 tmux 시작

```bash
tmux new -s awm-demo
```

### 2. pane을 나누고 에이전트 실행

```bash
tmux split-window -h
tmux split-window -v
tmux select-layout tiled
```

예를 들면 이렇게 배치합니다.

```text
codex     claude
gemini    opencode
```

각 pane에서 해당 에이전트를 실행합니다. 예를 들어 Codex pane에서는 `codex`, Claude pane에서는 `claude`, OpenCode pane에서는 `opencode`처럼 실행합니다. 실제 명령은 본인 환경에 설치된 CLI 이름을 쓰면 됩니다.

### 3. pane 이름 붙이기

각 pane 안에서 이름을 붙입니다.

```bash
tmux select-pane -T codex
```

다른 pane에서는 `claude`, `gemini`, `opencode`처럼 바꿔서 실행합니다.

에이전트에게 말로 시켜도 됩니다.

```text
이 tmux pane 이름을 codex로 바꿔줘.
```

pane 이름은 handoff를 보낼 주소처럼 쓰입니다. 이름이 단순해야 덜 헷갈립니다.

### 4. tmux handoff 켜기

작업을 보내는 쪽 에이전트에게 말합니다.

```text
tmux handoff on
```

이 명령을 말하기 전까지는 tmux 전용 설명을 불러오지 않습니다. 일반 세션은 가볍게 유지하고, tmux가 필요할 때만 켜는 방식입니다.

### 5. 실제로 작업 넘기기

예를 들어 Codex에게 이렇게 말할 수 있습니다.

```text
로그인만 되는 웹사이트를 설계해줘. 설계가 끝나면 tmux pane claude로 리뷰 handoff를 보내줘.
```

Claude 리뷰가 돌아오면 다시 Codex에게:

```text
Claude 리뷰를 반영하고, 구현은 tmux pane opencode로 넘겨줘.
```

이때 tmux는 단지 전달 채널입니다. 실제 기록은 항상 `AIMemory/work.log`와 `AIMemory/handoff_*.md`에 남습니다. 그래서 pane이 닫히거나 세션이 바뀌어도 handoff 기록은 프로젝트 안에 남아 있습니다.

### 6. 연결 테스트

pane 이름이 맞는지 먼저 확인하고 싶으면 high-five 테스트를 씁니다.

```text
tmux pane claude에 high-five를 보내고, 내 source pane으로 HIGHFIVE_CONFIRMED를 돌려줘.
```

이 테스트는 tmux 전달만 확인합니다. handoff 파일을 만들거나 `work.log`를 수정하지 않습니다. 보낸 쪽은 전송 뒤 두 pane을 자동 확인하지 않고,
`HIGHFIVE_CONFIRMED`를 다시 보내 달라고 재요청하지 않습니다. 일부 터미널 UI에서는 현재 턴이 끝나거나 사용자가 Enter를 누르거나 화면이 다시 그려질 때까지 반환 프롬프트가 보이지 않을 수 있으므로, receiver는 source pane에 tmux status-line 알림도 함께 띄웁니다.

### 7. tmux handoff 끄기

현재 에이전트 세션에서 tmux 전달을 끄려면:

```text
tmux handoff off
```

이후에는 `tmux-pane:<name>` 같은 대상이 있어도 일반 AICP handoff 파일 흐름만 사용합니다. 다시 쓰려면 `tmux handoff on`을 말하면 됩니다.

---

## 설치하면 어떤 파일이 생기나요?

```text
your-project/
├─ your-code/
└─ AIMemory/
   ├─ INDEX.md
   ├─ PROJECT_OVERVIEW.md
   ├─ PROTOCOL.md
   ├─ work.log
   ├─ archive/
   ├─ cold/
   └─ handoff_*.md
```

| 파일 | 역할 |
| --- | --- |
| `INDEX.md` | 무엇을 어디서 찾아야 하는지 알려주는 색인 |
| `PROJECT_OVERVIEW.md` | 새 에이전트가 프로젝트를 빠르게 이해하기 위한 요약 |
| `PROTOCOL.md` | 에이전트들이 따라야 하는 작업 기억 규칙 |
| `work.log` | 최근 명령, 작업, 결정, 테스트, handoff 기록 |
| `archive/` | 오래된 작업 로그 |
| `cold/` | 긴 기간의 요약 기록 |
| `handoff_*.md` | 에이전트 사이의 작업 인수인계 문서 |

tmux를 사용할 때만 선택적으로 `AIMemory/tmux-handoff.md`가 생길 수 있습니다. 일반 설치에서는 꼭 필요하지 않습니다.

---

## 좋은 사용 예시

- Codex가 기능 설계를 하고 Claude가 리뷰할 때
- Claude가 리뷰한 내용을 OpenCode가 구현할 때
- Gemini에게 분석이나 검증만 맡기고 싶을 때
- `/compact`, 모델 변경, 세션 종료 뒤에도 작업 맥락을 잃고 싶지 않을 때
- 여러 에이전트가 같은 프로젝트에서 동시에 일하지만 서로의 작업을 밟지 않게 하고 싶을 때

---

## 한 줄 요약

```text
여러 AI 에이전트를 열어두고, 작업 기억은 AIMemory 하나로 공유하세요.
```

`agent-work-mem`은 AI 코딩 에이전트들을 느슨하지만 기록 가능한 팀으로 묶어주는 markdown-first 프로토콜입니다.
