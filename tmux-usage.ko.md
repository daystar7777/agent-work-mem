# agent-work-mem tmux 사용법

[README.ko.md](README.ko.md) | [English README](README.md)

> Codex, Claude, Gemini, OpenCode 같은 AI 코딩 에이전트를 tmux pane에 동시에 띄우고, `agent-work-mem` handoff로 작업을 넘기는 입문용 가이드입니다.

---

## 한 줄 요약

```text
tmux는 전달 채널이고, AIMemory는 작업 기록입니다.
```

tmux는 여러 터미널 pane을 한 화면에 띄우는 도구입니다. `agent-work-mem`은 각 에이전트가 읽고 쓰는 공유 작업 기억입니다.

둘을 같이 쓰면 이런 흐름이 됩니다.

```text
Codex가 설계 -> Claude가 리뷰 -> Codex가 반영 -> OpenCode가 구현
```

tmux는 다른 pane에 프롬프트를 붙여 넣어 작업을 시작하게 해줍니다. 하지만 진짜 기록은 항상 프로젝트 안의 `AIMemory/work.log`와 `AIMemory/handoff_*.md`에 남습니다.

---

## 준비물

- tmux가 실행되는 터미널 환경
- 같은 프로젝트 폴더에서 실행되는 AI 코딩 에이전트들
- 프로젝트에 설치된 `agent-work-mem`

설치가 아직 안 되어 있으면 프로젝트 폴더에서 아무 에이전트에게 이렇게 말하세요.

```text
Install daystar7777/agent-work-mem into this project.
```

설치가 끝나면 `AIMemory/` 폴더가 생깁니다.

---

## 1. tmux 세션 시작

프로젝트 폴더에서 tmux 세션을 시작합니다.

```bash
tmux new -s awm-demo
```

세션 이름은 아무거나 괜찮습니다. 예시에서는 `awm-demo`를 사용합니다.

---

## 2. pane 나누기

한 화면에 여러 에이전트를 띄울 수 있게 pane을 나눕니다.

```bash
tmux split-window -h
tmux split-window -v
tmux select-layout tiled
```

예시 배치:

```text
codex     claude
gemini    opencode
```

pane 사이 이동은 보통 `Ctrl-b`를 누른 뒤 방향키를 누르면 됩니다.

---

## 3. 각 pane에서 에이전트 실행

각 pane에서 원하는 에이전트를 실행합니다.

예시:

```bash
codex
```

```bash
claude
```

```bash
gemini
```

```bash
opencode
```

실제 명령어 이름은 본인 환경에 설치된 CLI에 맞춰 사용하면 됩니다.

---

## 4. pane 이름 붙이기

handoff를 보낼 때 pane 이름을 주소처럼 사용합니다. 이름을 짧고 명확하게 붙이는 것이 좋습니다.

각 pane 안에서 다음 명령을 실행합니다.

```bash
tmux select-pane -T codex
```

다른 pane에서는 이름만 바꿔 실행합니다.

```bash
tmux select-pane -T claude
tmux select-pane -T gemini
tmux select-pane -T opencode
```

에이전트에게 말로 시켜도 됩니다.

```text
이 tmux pane 이름을 codex로 바꿔줘.
```

---

## 5. 각 에이전트가 AIMemory를 읽게 하기

새로 열린 에이전트에게 먼저 프로젝트 기억을 읽게 합니다.

```text
Read the project structure and AIMemory, then tell me you understand the current state.
```

이렇게 하면 에이전트가 보통 다음 파일들을 읽습니다.

- `AIMemory/INDEX.md`
- `AIMemory/PROJECT_OVERVIEW.md`
- `AIMemory/work.log`

이 단계는 “지금 이 프로젝트가 어디까지 진행됐는지” 맞추는 과정입니다.

---

## 6. tmux handoff 켜기

작업을 보내는 쪽 에이전트에게 말합니다.

```text
tmux handoff on
```

중요한 점:

- 이 명령을 말하기 전에는 tmux 전용 설명을 읽지 않습니다.
- 현재 세션이 tmux 안인지 확인한 뒤에만 tmux handoff를 켭니다.
- tmux 밖이면 `tmux handoff unavailable`처럼 사용할 수 없다고 알려야 합니다.
- 켜진 뒤에도 tmux는 전달만 담당하고, 기록은 `AIMemory/`에 남습니다.

---

## 7. 먼저 연결 테스트하기

실제 작업을 넘기기 전에 high-five 테스트로 pane 이름이 맞는지 확인할 수 있습니다.

```text
tmux pane claude에 high-five를 보내고, 내 source pane으로 HIGHFIVE_CONFIRMED를 돌려줘.
```

이 테스트는 tmux 전달만 확인합니다.

- handoff 파일을 만들지 않습니다.
- `work.log`를 수정하지 않습니다.
- source pane은 전송 뒤 두 pane을 자동 확인하지 않고, 같은 receiver에게 `HIGHFIVE_CONFIRMED` 재전송을 요청하지 않습니다.
- 일부 터미널 UI에서는 현재 턴이 끝나거나 사용자가 Enter를 누르거나 화면이 다시 그려질 때까지 반환 프롬프트가 보이지 않을 수 있으므로, receiver는 source pane에 tmux status-line 알림도 함께 띄웁니다.
- pane 이름을 잘못 붙였는지 빠르게 확인할 수 있습니다.

---

## 8. 실제 작업 넘기기

예를 들어 Codex pane에서 설계를 맡기고, Claude pane에 리뷰를 넘길 수 있습니다.

```text
로그인만 되는 웹사이트를 설계해줘.
설계가 끝나면 tmux pane claude로 리뷰 handoff를 보내줘.
```

이때 Codex는 보통 다음 일을 합니다.

1. 설계 내용을 정리합니다.
2. `AIMemory/handoff_*.md` 파일을 만듭니다.
3. `AIMemory/work.log`에 handoff 기록을 남깁니다.
4. tmux를 통해 Claude pane에 handoff 안내 프롬프트를 붙여 넣습니다.

Claude는 handoff를 읽고 리뷰한 뒤, 다시 결과 handoff를 남깁니다.

---

## 9. 리뷰 반영 후 구현 넘기기

Claude 리뷰가 돌아오면 Codex에게 이렇게 말할 수 있습니다.

```text
Claude 리뷰를 반영해줘.
그다음 구현은 tmux pane opencode로 넘겨줘.
```

그러면 OpenCode가 구현 담당 pane이 됩니다.

이 흐름에서 역할은 이렇게 나뉩니다.

| 역할 | 담당 예시 |
| --- | --- |
| 설계 | Codex |
| 리뷰 | Claude |
| 추가 검토 | Gemini |
| 구현 | OpenCode |

정해진 규칙은 아닙니다. 본인이 원하는 에이전트 조합으로 바꾸면 됩니다.

---

## 10. tmux handoff 끄기

현재 에이전트 세션에서 tmux 전달을 끄려면:

```text
tmux handoff off
```

끄고 나면 이후 handoff는 일반 AICP 파일 흐름만 사용합니다. 즉, `AIMemory/handoff_*.md`와 `work.log`를 통해 이어받습니다.

다시 켜려면:

```text
tmux handoff on
```

---

## 자주 쓰는 tmux 명령

| 목적 | 명령 |
| --- | --- |
| 새 세션 시작 | `tmux new -s awm-demo` |
| 가로 분할 | `tmux split-window -h` |
| 세로 분할 | `tmux split-window -v` |
| 타일 배치 | `tmux select-layout tiled` |
| 현재 pane 이름 변경 | `tmux select-pane -T codex` |
| pane 목록 확인 | `tmux list-panes -a` |
| 세션에서 잠시 나가기 | `Ctrl-b` 후 `d` |
| 세션 다시 들어가기 | `tmux attach -t awm-demo` |

---

## 문제 해결

### `tmux handoff on`이 안 됩니다

현재 에이전트가 tmux 안에서 실행 중인지 확인하세요.

```bash
echo "$TMUX"
```

값이 비어 있으면 tmux 안이 아닐 수 있습니다.

### handoff가 엉뚱한 pane으로 갑니다

pane 이름을 다시 확인하세요.

```bash
tmux list-panes -a -F '#{pane_id} #{pane_title} #{session_name}:#{window_index}.#{pane_index}'
```

그리고 각 pane에서 다시 이름을 붙이세요.

```bash
tmux select-pane -T claude
```

### pane은 받았는데 작업을 시작하지 않습니다

가끔 프롬프트가 붙여 넣어졌지만 Enter가 제대로 안 들어간 것처럼 보일 수 있습니다. 이 경우 target pane에서 Enter를 한 번 눌러 보세요.

그래도 안 되면 `AIMemory/work.log`와 최신 `AIMemory/handoff_*.md`를 직접 열어 이어받으면 됩니다. tmux가 실패해도 기록은 남아 있어야 합니다.

### 기록이 남지 않았습니다

tmux high-five 테스트는 기록을 남기지 않는 것이 정상입니다. 실제 handoff에서만 `AIMemory/handoff_*.md`와 `work.log` 기록이 생깁니다.

---

## 추천 첫 실습

처음에는 작은 작업으로 테스트하세요.

```text
Codex야, 로그인만 되는 웹사이트를 간단히 설계해줘.
설계가 끝나면 tmux pane claude로 리뷰 handoff를 보내줘.
Claude 리뷰가 오면 반영하고, 구현은 tmux pane opencode로 넘겨줘.
```

이 실습 하나로 다음 흐름을 모두 확인할 수 있습니다.

- `AIMemory/` 설치
- 에이전트별 프로젝트 기억 읽기
- tmux pane 이름 지정
- `tmux handoff on`
- 리뷰 handoff
- 구현 handoff
- `work.log` 기록 확인

---

## 기억할 점

- tmux는 화면과 전달을 편하게 해주는 도구입니다.
- AIMemory가 실제 작업 기억입니다.
- handoff 파일은 사람이 읽을 수 있는 markdown입니다.
- `tmux handoff on`은 필요할 때만 켭니다.
- `tmux handoff off`로 현재 세션에서 끌 수 있습니다.
- pane 이름은 짧고 명확하게 붙이세요.

헷갈리면 이 문장만 기억하면 됩니다.

```text
tmux로 보내고, AIMemory에 남긴다.
```
