---
name: flowsmith-run
description: flowsmith のフロー定義（YAML）を読み込み、ステップを順に subagent へ派発して実行するエンジン。中断後の再実行では state.json を検出して続きから再開する。「/flowsmith-run <flow.yaml>」と指定されたとき、またはフロー定義の実行・再開を頼まれたときに使う。引数はフロー定義 YAML のパス。
---

# flowsmith 実行エンジン (v2)

フロー定義 YAML を受け取り、宣言されたステップを順に実行する。
**エンジンは推測で進まない。** 解釈に迷う状況はすべて `seized`（停止・人間待ち）として報告する。
**状態はすべてディスクに置く。** セッションが死んでもフローは死なない。

## v2 の実装範囲

- ステップタイプ: `make`（`foreach` 行単位反復を含む）。`make-check` / `branch` / `loop` は未実装（検出したら seized）
- ゲート: `exists` / `sections`。他は「未評価ゲート」として完了報告に明示
- **断点再開: 実装済み**（step 単位 + foreach 行単位）
- ログ: `run.jsonl` に全イベント追記

## 実行時ファイル（詳細は docs/runtime.md）

```text
${OUT}/.flowsmith/
├── state.json   # 断点。step 完了ごと・foreach 行完了ごとに必ず更新
├── run.jsonl    # 流水ログ（追記のみ）
└── runs/        # 完了フローの state アーカイブ
```

---

## Phase 0: 初期化と検証

1. 引数の YAML を Read で読み込む。
2. `paths` 変数を展開（`OUT` は予約・必須。無ければ seized）。相対パスはカレントディレクトリ基準で絶対化する。
3. 検証（失敗 → seized）：`flow.id`・`steps[]` 存在 / step id 一意 / type 既知 / `next` 参照先実在 / `brief` か `brief_file` のどちらか必須 / `foreach` があれば `table`・`key` 必須。
4. `${OUT}/.flowsmith/` を作成（無ければ）。

## Phase 0.5: 状態検出と再開判定

`${OUT}/.flowsmith/state.json` を確認する：

| 状況 | 動作 |
|------|------|
| 無い | 新規実行。state.json を作成し `flow_start` をログ |
| あり・`flow_id` 不一致 | **seized**（別フローの OUT 流用。上書き事故防止） |
| あり・`status: running` | **再開**。`completed_steps` をスキップして `next_step` から。foreach 中の step は `done_keys` 以外の行から。`flow_resume` をログし、ユーザーに `▶ 再開: {n} ステップをスキップ、{step_id} から続行` と明示 |
| あり・`status: seized` | 前回の停止理由を表示し、当該 step から再実行（人間が裁定済みとみなす） |
| あり・`status: complete`（異常系） | runs/ へ退避して新規実行 |

新規実行時は実行計画を 1 行ずつ提示する：`[順序] step_id — title`。

## Phase 1: ステップ実行ループ

### 1.1 進捗表示とログ

`[完了数/総数] {step_id}: {title}` を表示し、`step_start` を run.jsonl に追記。

### 1.2 派発準備

`brief_file` 指定なら Read で読み込む。`refs` / `inputs` / `outputs` を絶対パス化。

### 1.3 実行（通常 step）

派発プロンプト雛形 A を埋め、Agent ツール（general-purpose）で subagent を起動する。

### 1.3' 実行（foreach step）

1. `foreach.table` のファイルを Read し、Markdown 表から `foreach.key` 列を抽出して行キー一覧を作る（キーはファイル名に安全な形に正規化：`/` → `_` 等）。
2. state の当該 step に `foreach: {total, done_keys, pending_keys}` を初期化（再開時は既存の done_keys を保持）。
3. **pending の各行について順に**：
   - 雛形 B（行コンテキスト付き）で派発。出力先は `{outputs[].path}/{行キー}.md`
   - 結果ブロック検証（1.4 と同じ）
   - **行完了ごとに state.json の done_keys を更新**し、`row_end` をログ ← 行単位再開の根拠
4. 全行完了で step 完了。

### 1.4 結果ブロックの解析

subagent 最終メッセージの ` ```flowsmith-result ` フェンスを抽出：

- 欠落／YAML 不正／`step` 不一致 → 「結果ブロック欠落。形式厳守」と添えて **1 回だけ再派発**。再失敗 → seized
- `status: seized` → seized 処理（subagent の notes を引用）
- `status: done` → 次へ

### 1.5 ゲート評価（エンジン自身が決定的に判定。LLM に聞かない）

| gate | 判定方法 |
|------|----------|
| `exists: <path>` | Glob/Read でファイル実在確認 |
| `sections: [...]` | Grep で各見出し文字列の存在確認 |

不合格 → 一覧を添えて **1 回だけ再派発**（`gate_fail` をログ）。再不合格 → seized。
未実装 gate は評価せず完了報告で明示。

### 1.6 step 完了処理

state.json を更新（`completed_steps` 追加 / `next_step` 設定 / steps.{id} に status・rounds・outputs・notes・finished_at）。
`step_end` をログ（status / rounds / duration_s / **agent_tokens** = 派発結果に表示される subagent トークン量）。

### 1.7 ルーティング

`next` 指定があればその step へ、無ければ配列の次へ。末尾完了 → Phase 2 へ。

## Phase 2: 完了処理と報告

1. state.json を `status: complete` に更新 → `runs/{YYYYMMDD-HHmmss}_{flow_id}.json` として保存 → state.json を削除
2. `flow_complete` をログ（total_steps / total_agent_tokens）
3. ユーザーへ報告：実行 step 一覧（再派発有無）／成果物パス一覧／合計 agent_tokens／未評価ゲート・警告

## seized 時の処理（フォーマット厳守）

1. state.json を `status: seized`、`seized: {step, reason}` に更新し、`seized` をログ
2. 以下を報告して**停止**（勝手なリトライ・回避をしない）：

```
⛔ flowsmith seized — 人間の裁定が必要です
- フロー: {flow.id}
- ステップ: {step_id}（{title}）
- 原因: {具体的に}
- 経緯: {再派発の有無、subagent の notes 等}
- 復旧の選択肢: {brief 修正して再実行 / ゲート緩和 / 成果物を手動修正 など}
- 再開方法: 同じコマンドを再実行（このステップから再開されます）
```

---

## 派発プロンプト雛形 A（通常 step）

```
あなたは flowsmith フローの 1 ステップを実行する作業者です。担当範囲外のことはしないでください。

# ステップ: {title}（{step_id}）

## 作業指示
{brief}

## 参照資料（作業前に Read で読むこと）
{refs 箇条書き／なし}

## 入力
- {id}: {絶対パス} — {note}

## 出力（必ずこの絶対パスに書くこと。これ以外のファイルを変更しない）
- {id}: {絶対パス}

## 完了時の約束（厳守）
最終メッセージの末尾に、次の形式のフェンスブロックを必ず含めること：

```flowsmith-result
step: {step_id}
status: done
outputs:
  - {実際に書いたパス}
notes: {一行メモ}
```

完遂できない場合も省略せず、status: seized とし notes に理由を書くこと。
```

## 派発プロンプト雛形 B（foreach の 1 行）

雛形 A に以下を追加する：

```
## 今回の担当行（この 1 行だけを処理すること）
{foreach.key の各列名}: {行の値}

## 出力（この行専用のパス）
- {id}: {outputs[].path}/{正規化済み行キー}.md
```

結果ブロックの `step` は `{step_id}#{行キー}` とする。
