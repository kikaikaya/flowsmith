# flowsmith

AI エージェントのワークフローを **1 つの YAML** で宣言し、**中断しても続きから動かす** ための Claude Code 向けオーケストレーション Skill。

> flow（ワークフロー）を smith（鍛冶屋）のように打って仕上げる、という名前です。

## 設計の柱

- **単一ファイル定義** — 1 フロー = 1 YAML。全体像は常に 1 ファイルで掴める
- **チェックポイント再開** — 状態はすべてディスクに置く。セッションが死んでもフローは死なない（反復処理は行単位で再開）
- **二層検査** — 機械ゲート（決定的検査）が先、LLM レビューは後。安く確実に落とせるものを LLM に回さない
- **行き詰まったら止まる** — 解析不能・再試行上限超過は推測で進まず停止し、人間に裁定を返す
- **コスト可観測** — ステップごとの所要時間・トークンをログに残す

## Quick Start

```bash
# 1. インストール（skills/ を ~/.claude/skills/ にコピー、またはプラグインとして導入）

# 2. リポジトリのルートで Claude Code を起動し、サンプルフローを実行
claude
> /flowsmith-run examples/hello/flow.yaml
```

2 ステップの最小フロー（ファイル一覧の作成 → サマリー生成）が順に実行され、
成果物が `flowsmith-out/` に出力されます。

フロー定義の書き方は [docs/schema.md](docs/schema.md) を参照。

## ステータス

🚧 **v5。** 実装済み: `make` / `make-check`（maker-checker レビューループ）/ `branch` / `loop` の
4 ステップタイプ、機械ゲート 5 種、チェックポイント再開（ステップ単位 + 行単位）、
熔断（max_rounds / max_laps / max_returns）、フロー目的の全エージェント共有（goal）、
臨時質問（ask）。実ワークロードでの検証は進行中。

最初の実用ユースケースは [migration-factory](https://github.com/KIKAIKAYA/migration-factory) の
survey-unit（レガシー現状調査の自動化）です。

## License

MIT © kikaikaya
