# 実行時ファイル仕様 — state / log / archive

> flowsmith の「中断しても続きから動く」を支えるファイル群。すべて `${OUT}/.flowsmith/` に置かれる。

```text
${OUT}/.flowsmith/
├── state.json     # 断点（現在の実行状態。完了時に削除される）
├── run.jsonl      # 流水ログ（追記のみ）
└── runs/          # 完了したフローの state アーカイブ
    └── 20260611-153000_hello.json
```

## 1. state.json

```json
{
  "flow_id": "hello",
  "flow_file": "examples/hello/flow.yaml",
  "status": "running",
  "started_at": "2026-06-11 15:00:00",
  "updated_at": "2026-06-11 15:04:00",
  "completed_steps": ["inventory"],
  "next_step": "digest",
  "steps": {
    "inventory": {
      "status": "done",
      "rounds": 1,
      "outputs": ["flowsmith-out/01-files.md"],
      "notes": "7 ファイルを表にまとめた",
      "finished_at": "2026-06-11 15:02:00"
    },
    "write_rows": {
      "status": "in_progress",
      "foreach": {
        "total": 6,
        "done_keys": ["binary_search", "quicksort", "bfs"],
        "pending_keys": ["dijkstra", "union_find", "dp"]
      }
    }
  },
  "seized": null
}
```

| フィールド | 説明 |
|------------|------|
| `status` | `running` / `asking` / `seized` / `complete` |
| `completed_steps` | 完了した step id（順序どおり） |
| `next_step` | 次に実行すべき step id |
| `steps.{id}.foreach.done_keys` | **行単位の完了キー**。再開時はここに無い行だけ実行する |
| `steps.{id}.qa` | 臨時質問の履歴 `[{ "q": ..., "a": ... }]`（a が null = 未回答） |
| `steps.{id}.asks` | 当該 step の質問回数（max_asks 熔断の根拠） |
| `seized` | seized 時のみ `{ "step": ..., "reason": ... }` |

**更新タイミング**：① flow 開始時に作成 ② **各 step 完了ごと** ③ **foreach の各行完了ごと**（ここが行単位再開の根拠）④ seized 時 ⑤ 完了時にアーカイブして削除。

## 2. 再開プロトコル

エンジン起動時に `${OUT}/.flowsmith/state.json` を確認する：

| 状況 | 動作 |
|------|------|
| 無い | 新規実行 |
| あり・`flow_id` が今回と不一致 | **seized**（別フローの OUT を流用している。上書き事故防止） |
| あり・`status: running` | **再開**：`completed_steps` をスキップし `next_step` から続行。foreach 中なら `done_keys` 以外の行から |
| あり・`status: asking` | **未回答の質問を再提示**：qa の a が null の質問をユーザーに見せ、回答を得て当該 step を再派発 |
| あり・`status: seized` | 前回の停止理由を表示したうえで、当該 step から再実行（人間が裁定済みとみなす） |
| あり・`status: complete`（異常系） | runs/ へ退避して新規実行 |

再開時は必ずユーザーに明示する：`▶ 再開: {n} ステップをスキップ、{step_id} から続行`。

## 3. run.jsonl（流水ログ・追記のみ）

1 行 = 1 イベント：

```jsonl
{"event":"flow_start","flow":"hello","ts":"2026-06-11 15:00:00"}
{"event":"step_start","step":"inventory","ts":"2026-06-11 15:00:05"}
{"event":"step_end","step":"inventory","status":"done","rounds":1,"duration_s":55,"agent_tokens":28259,"ts":"2026-06-11 15:01:00"}
{"event":"row_end","step":"write_rows","key":"quicksort","status":"done","agent_tokens":9100,"ts":"..."}
{"event":"gate_fail","step":"digest","gates":["sections: サマリー"],"ts":"..."}
{"event":"flow_resume","flow":"hello","skipped":["inventory"],"ts":"..."}
{"event":"seized","step":"digest","reason":"結果ブロック不正 ×2","ts":"..."}
{"event":"flow_complete","flow":"hello","total_steps":2,"total_agent_tokens":46172,"ts":"..."}
```

| イベント | 必須フィールド |
|----------|----------------|
| `flow_start` / `flow_resume` | flow（resume は skipped も） |
| `step_start` / `step_end` | step（end は status / rounds / duration_s / agent_tokens） |
| `row_end` | step / key / status / agent_tokens |
| `gate_fail` | step / gates |
| `ask` | step / questions（回答時は同 step の `answer` イベントで a を記録） |
| `seized` | step / reason |
| `flow_complete` | flow / total_steps / total_agent_tokens |

`agent_tokens` は派発結果に表示される subagent のトークン使用量（コスト可観測の根拠データ）。

## 4. runs/（アーカイブ）

フロー完了時、state.json を `runs/{YYYYMMDD-HHmmss}_{flow_id}.json`（`status: complete` に更新したもの）として保存し、state.json を削除する。過去の実行履歴は run.jsonl と runs/ で追跡できる。
