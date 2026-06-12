---
name: flowsmith-run
description: flowsmith のフロー定義（YAML）を読み込み、ステップを順に subagent へ派発して実行するエンジン。中断後の再実行では state.json を検出して続きから再開する。「/flowsmith-run <flow.yaml>」と指定されたとき、またはフロー定義の実行・再開を頼まれたときに使う。引数はフロー定義 YAML のパス。
---

# flowsmith 実行エンジン (v5)

フロー定義 YAML を受け取り、宣言されたステップを順に実行する。
**エンジンは推測で進まない。** 解釈に迷う状況はすべて `seized`（停止・人間待ち）として報告する。
**状態はすべてディスクに置く。** セッションが死んでもフローは死なない。
**合否・路由はエンジンが機械的に決める。** LLM の自己申告を合否判定に使わない。

## v4 の実装範囲

- ステップタイプ: `make`（foreach 行単位反復）/ `make-check`（maker-checker レビューループ）/ **`branch`（二者択一の判定分岐）/ `loop`（条件付き反復）**
- ゲート: `exists` / `sections` / `min_lines` / `table_rows_min` / `count_match`
- 断点再開: step 単位 + foreach 行単位
- 失敗履歴: 繰り返し FAIL する観点を再派発時に明示注入
- 熔断: max_rounds / max_laps 超過 → seized + 行き詰まりサマリー
- **carry: 判定理由を戻り先ステップへ引き継ぐ**
- **ask: 臨時質問（★v5）— 指示が曖昧なとき agent が推測せず人間に質問し、回答を得て続行する**

## 実行時ファイル（詳細は docs/runtime.md）

```text
${OUT}/.flowsmith/
├── state.json   # 断点。step 完了・foreach 行完了・レビューラウンドごとに必ず更新
├── run.jsonl    # 流水ログ（追記のみ）
└── runs/        # 完了フローの state アーカイブ
${OUT}/reviews/  # checker のレビュー表（{step_id}_r{n}.md）
```

---

## Phase 0: 初期化と検証

1. 引数の YAML を Read で読み込む。
2. `paths` 変数を展開（`OUT` は予約・必須。無ければ seized）。相対パスはカレントディレクトリ基準で絶対化。
3. 検証（失敗 → seized）：`flow.id`・`steps[]` 存在 / step id 一意 / type 既知 / `next` 参照先実在 / `brief` か `brief_file` 必須（make-check は `maker.brief` と `checker.viewpoints` 必須）/ `foreach` は `table`・`key` 必須。
4. `${OUT}/.flowsmith/` を作成。

## Phase 0.5: 状態検出と再開判定

`${OUT}/.flowsmith/state.json` を確認：

| 状況 | 動作 |
|------|------|
| 無い | 新規実行。state.json 作成、`flow_start` ログ |
| あり・`flow_id` 不一致 | **seized**（別フローの OUT 流用） |
| あり・`status: running` | **再開**：completed_steps をスキップ、next_step から。foreach は done_keys 以外の行から。`flow_resume` ログ + ユーザーに明示 |
| あり・`status: seized` | 前回の停止理由を表示し、当該 step から再実行（fail_history は引き継ぐ） |
| あり・`status: complete`（異常系） | runs/ へ退避して新規実行 |

新規実行時は実行計画を提示：`[順序] step_id — title (type)`。

## Phase 1: ステップ実行ループ

### 1.1 進捗とログ

`[完了数/総数] {step_id}: {title}` 表示、`step_start` を追記。

### 1.2 派発準備

`brief_file` は Read で読み込み。`refs` / `inputs` / `outputs` を絶対パス化。

**ゲート要件の brief 注入（★v5.1）**：step に `sections` ゲートがある場合、maker への
派発プロンプトに次のセクションを必ず追加する（実負荷で「確認が必要な事項」と書いて
「確認事項」ゲートに落ちた事故への対策。機械ゲートは完全一致で判定するため、
要求見出しを作業者に一字一句伝える）：

```
## 出力の必須見出し（一字一句この通りに書くこと。表記の言い換え禁止）
- {sections の各見出し}
```

### 1.3 実行（make）

雛形 A で派発 → 結果ブロック解析（1.5）→ ゲート評価（1.6）→ 完了処理（1.8）。

### 1.3' 実行（make + foreach）

1. `foreach.table` を Read し、表から `key` 列で行キー一覧を作る（`/` → `_` 等に正規化）。
2. state に `foreach: {total, done_keys, pending_keys}` を初期化（再開時は done_keys 保持）。
3. pending の各行：雛形 A + 行コンテキストで派発 → 結果ブロック解析 → **行ごとに state 更新 + `row_end` ログ**。
4. 全行完了で step 完了。

### 1.3'' 実行（make-check）★v3

```text
round = state の続き（新規なら 1）
[1] maker 派発（雛形 A。round ≥ 2 なら「前回の指摘」セクションを追加）
[2] ゲート評価（機械）— 不合格 → 一覧付きで maker 1 回だけ再派発、再不合格 → seized
    （ゲート再試行はラウンドを消費しない）
[3] checker 派発（雛形 C）→ レビュー表 + verdicts 入り結果ブロック
[4] エンジンが pass_when を verdicts に機械適用：
    - no_high_fail: severity high の fail が 0 件 → 合格
    - all_pass:     fail が 0 件 → 合格
[5] 合格 → 完了処理へ。`check_end` ログ
[6] 不合格 → fail した観点を state の fail_history に加算、`check_end` ログ、round += 1
    - round ≤ max_rounds → [1] へ（fail 観点の修正指示 + レビュー表パスを注入。
      fail_history で 2 回以上の観点は「繰り返し指摘」として強調）
    - round > max_rounds → 【熔断】seized。行き詰まりサマリーを報告
```

**熔断時の行き詰まりサマリー**（seized 報告に含める）：
- 通らない観点（id / severity / 各ラウンドの根拠）
- 試行ラウンド数と各ラウンドの修正内容（maker の notes）
- レビュー表のパス一覧
- 人間への選択肢：観点を緩める / brief を直す / 成果物を手動修正して再実行

### 1.3''' 実行（branch）★v4

```text
[1] arbiter 派発（雛形 D）。判定材料 = uses に列挙された上流 step の outputs と state 結果摘要
[2] 結果ブロックの status（yes / no）と reason を取得
[3] エンジンが route.yes / route.no で次ステップを決定（status が yes/no 以外 → 1.5 の再派発規則）
[4] carry が定義され、かつ採られた路由先が carry.to と一致する場合：
    reason を state の steps.{carry.to}.carried に保存。
    次回その step を派発するとき「制御からの引き継ぎ」セクションとして注入する
[5] state 更新（steps.{id}: status / reason）、`route` ログ（from / verdict / to）
```

**戻りジャンプの扱い**：route 先が実行済み step の場合、その step を再実行対象に戻す
（completed_steps から外し、make-check なら rounds・fail_history は引き継ぐ）。

**戻りジャンプの熔断**：branch の戻りジャンプ回数を state（steps.{id}.returns）で数え、
`max_returns`（省略時 3）を超えたら **seized + 行き詰まりサマリー**（各回の reason 一覧）。
無限ループは構造的に起こり得ないようにする。

### 1.3'''' 実行（loop）★v4

branch と同じ仕組みで、status は `done` / `again`：

- `done` → route.done へ
- `again` → route.again へ（laps += 1 を state に記録）
- laps > `max_laps` → **熔断**: seized + 行き詰まりサマリー（各 lap の reason 一覧）

### 1.4 臨時質問（ask）の処理 ★v5

agent が `status: ask` を返したときの動き（社員の「ここ不明なので確認させてください」に相当）：

```text
[1] 結果ブロックの questions（1〜3 件）を取得
[2] state を status: asking に更新、questions を steps.{id}.qa に保存、`ask` ログ
[3] エンジンがその場でユーザーに質問を提示して回答を待つ（フローは中断しない。会話内で完結）
[4] 回答を qa に追記し、同じ step を再派発。プロンプトに「質疑応答」セクションを注入：

    ## 質疑応答（人間からの回答。指示の一部として扱うこと）
    Q1: {質問} → A1: {回答}

[5] state を running に戻して続行
```

- **seized との違い**：seized = 異常停止・人間の裁定待ち。ask = 作業前・作業中の確認。回答さえ得れば即続行する正常系。
- **濫用防止**：同一 step の質問回数を state（steps.{id}.asks）で数え、`max_asks`（省略時 2）超過 → seized（質問を繰り返すのは brief 自体に欠陥がある兆候。人間が brief を直すべき）。
- **回答待ち中にセッションが死んだ場合**：state は `asking` のまま残る。再開時に未回答の質問を再提示する。
- agent への指針はテンプレート側に記載：「推測で進めるな、致命的な曖昧さは ask で確認しろ。ただし些細なことは常識で補え」。

### 1.5 結果ブロックの解析

` ```flowsmith-result ` フェンスを抽出。欠落／不正／step 不一致 → 1 回だけ再派発 → 再失敗で seized。`status: seized` → seized 処理。`status: ask` → 1.4 の処理。

### 1.6 ゲート評価（エンジン自身が決定的に判定。LLM に聞かない）

| gate | 判定方法 |
|------|----------|
| `exists: <path>` | ファイル実在（Glob/Read） |
| `sections: [...]` | 各見出し文字列の存在（Grep） |
| `min_lines: <n>` | Read して行数 ≥ n |
| `table_rows_min: <n>` | 出力内 Markdown 表のデータ行（ヘッダ・罫線除く）≥ n |
| `count_match: {ref, tolerance}` | 出力の表データ行数が ref ファイルの表データ行数 ±tolerance |

不合格 → 一覧を添えて 1 回だけ再派発（`gate_fail` ログ）。再不合格 → seized。

### 1.7 ルーティング

`next` → その step へ / 無ければ配列の次へ / 末尾完了 → Phase 2。

### 1.8 step 完了処理

state 更新（completed_steps / next_step / steps.{id}: status・rounds・outputs・notes・fail_history・finished_at）。
`step_end` ログ（status / rounds / duration_s / agent_tokens）。

## Phase 2: 完了処理と報告

1. state を `complete` に → `runs/{YYYYMMDD-HHmmss}_{flow_id}.json` 保存 → state.json 削除
2. `flow_complete` ログ（total_steps / total_agent_tokens）
3. 報告：step 一覧（ラウンド数・再派発有無）／成果物／合計 agent_tokens／未評価ゲート

## seized 時の処理（フォーマット厳守）

1. state を `status: seized`、`seized: {step, reason}` に更新、`seized` ログ
2. 報告して**停止**（勝手なリトライをしない）：

```
⛔ flowsmith seized — 人間の裁定が必要です
- フロー / ステップ / 原因 / 経緯（熔断時は行き詰まりサマリー）
- 復旧の選択肢: …
- 再開方法: 同じコマンドを再実行（このステップから再開）
```

---

## 派発プロンプト雛形 A（maker / make）

```
あなたは flowsmith フローの 1 ステップを実行する作業者です。担当範囲外のことはしないでください。

## 作業背景（フロー全体の目的）
{flow.goal — 省略時はこのセクションごと省く}

# ステップ: {title}（{step_id}）

## 作業指示
{brief}

## 参照資料（作業前に Read で読むこと）
{refs／なし}

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

## 不明点の確認（ask）
指示に致命的な曖昧さがあり、推測で進めると成果物の方向を誤る場合は、
作業せずに次の形式で質問を返すこと（質問は 1〜3 件に絞る）：

```flowsmith-result
step: {step_id}
status: ask
questions:
  - {質問 1}
  - {質問 2}
notes: {なぜ確認が必要か一行}
```

ただし些細なこと（表記ゆれ・形式の微差など）は常識で補い、質問しないこと。
```

**round ≥ 2 の追加セクション**：

```
## 前回レビューの指摘（必ず対応すること）
レビュー表: {review path}（Read で読むこと）
- {V01} ({severity}): {修正指示}

## 繰り返し指摘されている観点（2 回以上 FAIL。今回最優先で対応）
- {V01}
```

**carry を受けた step の追加セクション**（state の carried がある場合）：

```
## ⚠ 差し戻し（前回の成果物は判定で不合格になった）
不合格の理由: {carried.reason}

上記の理由は「現状の説明」ではなく「解消すべき不足」である。
既存の成果物を修正し、この不足を必ず解消すること。「変更不要」という結論は禁止。
```

> 設計メモ: 当初は「判定理由を踏まえよ」という中立的な文面だったが、実験で maker が
> 理由文を「あるべき姿の説明」と誤読して無変更で返す事故が起きた。差し戻しであることを
> 明示し「変更不要は禁止」と書くことで誤読を塞ぐ。

**foreach の追加セクション**（結果ブロックの step は `{step_id}#{行キー}`）：

```
## 今回の担当行（この 1 行だけを処理すること）
{key 列名}: {値}

## 出力（この行専用のパス）
- {id}: {outputs[].path}/{正規化済み行キー}.md
```

## 派発プロンプト雛形 D（arbiter / branch・loop）★v4

```
あなたは flowsmith フローの判定担当（arbiter）です。**ファイルを変更してはいけません。判定だけを行います。**

## 作業背景（フロー全体の目的）
{flow.goal — 省略時はこのセクションごと省く}

# 判定: {title}（{step_id} / {branch または loop}）

## 判定基準
{arbiter.brief}

## 判定材料（Read で読むこと）
- {uses の各 step の outputs 絶対パス}
- {state の当該 step 結果摘要（rounds / notes 等）}

## 完了時の約束（厳守）
最終メッセージの末尾に、次の形式のフェンスブロックを必ず含めること：

```flowsmith-result
step: {step_id}
status: {branch は yes または no / loop は done または again}
reason: {判定理由。判定材料からの具体的根拠を 1〜3 行で。戻り先への指示になることを意識する}
```

判定基準そのものに致命的な曖昧さがあり判定できない場合は、status: ask とし
questions に確認事項を列挙すること（推測で判定しない）。
```

## 派発プロンプト雛形 C（checker）★v3

```
あなたは flowsmith フローの検査担当（checker）です。**成果物を修正してはいけません。検査と指摘だけを行います。**

## 作業背景（フロー全体の目的）
{flow.goal — 省略時はこのセクションごと省く}

# 検査: {title}（{step_id} / round {n}）

## 検査対象（Read で読むこと）
- {maker の outputs 絶対パス}

## 検査観点（これ以外の観点を追加しない）
- {V01} [{severity}]: {rule}
- {V02} [{severity}]: {rule}

## 作業手順
1. 検査対象を読む
2. 各観点を判定し、レビュー表を次のパスに書く: {OUT}/reviews/{step_id}_r{n}.md
   形式: | 観点 | 重大度 | 判定 | 根拠（引用） | 修正指示 |
   - 判定が pass の行は修正指示を「-」とする
   - 根拠は対象からの具体的な引用・行参照とする（印象で書かない）
3. 最終メッセージ末尾に結果ブロック（厳守）：

```flowsmith-result
step: {step_id}#check_r{n}
status: done
review: {レビュー表のパス}
verdicts:
  - {id: V01, severity: high, verdict: pass}
  - {id: V02, severity: mid, verdict: fail}
notes: {一行メモ}
```
```
