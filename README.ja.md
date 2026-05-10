# agent-work-mem

[English README](README.md) | [한국어 README](README.ko.md)

> 複数の AI コーディングエージェントを、1 つのチームのように使うためのファイルベース共有作業メモリです。
>
> Claude, Codex, Gemini, OpenCode, Cursor, Aider などのエージェントが、同じプロジェクトの `AIMemory/` を読み、作業を引き継ぎ、handoff できるようにします。

---

## なぜ必要ですか？

優秀な AI コーディングエージェントはたくさんあります。けれど、1 つだけではトークンや視点が足りないことがあります。一方で複数を使うと、毎回コンテキストをコピーして貼り付けるのが面倒です。

`agent-work-mem` は、プロジェクト内に `AIMemory/` という共有作業メモリを作ります。各エージェントは新しいセッションの開始時にこれを読み、作業中の判断や結果を markdown として残します。

たとえば、こういう流れが簡単になります。

```text
Codex が設計する -> Claude がレビューする -> OpenCode が実装する
```

サーバー、データベース、専用 SaaS は不要です。プロジェクト内の markdown ファイルだけで動きます。

---

## インストール

プロジェクトフォルダで任意のエージェントを開き、次のように伝えます。

```text
Fetch https://raw.githubusercontent.com/daystar7777/agent-work-mem/main/prompt.md and apply it to this project.
```

もっと短く言うなら:

```text
Install daystar7777/agent-work-mem into this project.
```

エージェントが `AIMemory/` フォルダを作り、共有作業メモリのプロトコルを設定します。

インストール後、別のエージェントを開くときは次のように始めます。

```text
Read the project structure and AIMemory, then tell me you understand the current state.
```

そのエージェントは `AIMemory/INDEX.md`, `AIMemory/PROJECT_OVERVIEW.md`, `AIMemory/work.log` を読み、現在の状態を理解します。

---

## 基本的な handoff

送る側のエージェントには、普通の言葉で伝えれば十分です。

```text
ログインだけできる Web サイトを設計してください。設計が終わったら Claude にレビュー handoff を作ってください。
```

受け取る側のエージェントには、次のように言います。

```text
最新の handoff を読んでレビューしてください。
```

エージェントは `AIMemory/handoff_*.md` を作り、`AIMemory/work.log` に誰がいつ何を渡したかを記録します。

handoff ファイルを人間が手で書く必要はありません。自然言語で頼むと、エージェントが構造化された handoff を作ります。

---

## tmux で複数エージェントを同時に使う

Codex, Claude, Gemini, OpenCode を同じプロジェクトで同時に開きたい場合は、tmux が一番わかりやすい方法です。画面を複数の pane に分け、それぞれに 1 つずつエージェントを起動します。

### 1. プロジェクトフォルダで tmux を開始

```bash
tmux new -s awm-demo
```

### 2. pane を分割してエージェントを起動

```bash
tmux split-window -h
tmux split-window -v
tmux select-layout tiled
```

例:

```text
codex     claude
gemini    opencode
```

各 pane で対応するエージェントを起動します。たとえば Codex の pane では `codex`、Claude の pane では `claude`、OpenCode の pane では `opencode` のように実行します。実際のコマンド名は、あなたの環境でインストールされている CLI 名に合わせてください。

### 3. pane に名前を付ける

各 pane の中で名前を付けます。

```bash
tmux select-pane -T codex
```

別の pane では `claude`, `gemini`, `opencode` のように変えて実行します。

エージェントに頼んでもかまいません。

```text
Rename this tmux pane to codex.
```

pane 名は handoff の宛先として使われます。短く、わかりやすい名前にするのがおすすめです。

### 4. tmux handoff を有効にする

作業を送る側のエージェントに次のように伝えます。

```text
tmux handoff on
```

このコマンドを言うまで、tmux 専用の説明は読み込まれません。通常のセッションは軽く保ち、tmux が必要なときだけ有効にします。

### 5. 実際に作業を渡す

たとえば Codex にこう頼めます。

```text
ログインだけできる Web サイトを設計してください。設計ができたら tmux pane claude にレビュー handoff を送ってください。
```

Claude のレビューが戻ったら、Codex にこう伝えます。

```text
Claude のレビューを反映し、実装は tmux pane opencode に渡してください。
```

tmux はあくまで配送チャネルです。真実の記録は常に `AIMemory/work.log` と `AIMemory/handoff_*.md` に残ります。そのため pane が閉じても、セッションやモデルが変わっても、handoff の履歴はプロジェクト内に残ります。tmux 配送は各プロンプトを一度だけ貼り付けて送信し、送信側や受信側が peer pane を自動確認したり検証用の Enter を追加送信したりしません。すぐ人間に見えるように、target/source pane には tmux status-line 通知も表示します。

### 6. 接続テスト

pane 名が正しく届くか確認したいときは、high-five テストを使います。

```text
Send a high-five to tmux pane claude; return HIGHFIVE_CONFIRMED to my source pane.
```

このテストは tmux の配送だけを確認します。handoff ファイルは作らず、`work.log` も変更しません。送信側は送信後に pane を自動確認せず、
`HIGHFIVE_CONFIRMED` の再送も依頼しません。一部の terminal UI では、現在のターンが終わる、ユーザーが Enter を押す、または画面が再描画されるまで返答プロンプトが見えないことがあるため、receiver は source pane に tmux status-line 通知も表示します。

### 7. tmux handoff を無効にする

現在のエージェントセッションで tmux 配送を止めるには:

```text
tmux handoff off
```

その後は `tmux-pane:<name>` のような宛先があっても、通常の AICP handoff ファイルの流れだけを使います。もう一度使う場合は `tmux handoff on` と伝えます。

---

## インストール後に作られるファイル

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

| ファイル | 役割 |
| --- | --- |
| `INDEX.md` | 必要な記録を探すための索引 |
| `PROJECT_OVERVIEW.md` | 新しいエージェントがプロジェクトをすばやく理解するための要約 |
| `PROTOCOL.md` | エージェントが従う共有作業メモリのルール |
| `work.log` | 最近の指示、作業、判断、テスト、handoff の記録 |
| `archive/` | 古い作業ログ |
| `cold/` | 長期間の要約 |
| `handoff_*.md` | エージェント間の引き継ぎ文書 |

tmux を使う場合だけ、必要に応じて `AIMemory/tmux-handoff.md` が作られることがあります。通常のインストールでは必須ではありません。

---

## よくある使い方

- Codex が設計し、Claude がレビューする
- Claude のレビューを反映して、OpenCode が実装する
- Gemini に分析や検証だけを任せる
- `/compact`, モデル変更, セッション終了の後でも作業文脈を失いたくない
- 複数のエージェントが同じプロジェクトで同時に作業しても、互いの作業を踏まないようにしたい

---

## ひとことで言うと

```text
複数の AI エージェントを開き、作業メモリは AIMemory で 1 つに共有します。
```

`agent-work-mem` は、AI コーディングエージェントをゆるく、しかし記録可能なチームとしてつなぐ markdown-first プロトコルです。
