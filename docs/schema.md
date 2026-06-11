# flowsmith フロー定義 schema — 草案 v0

> **設計原則**：1 フロー = 1 YAML ファイル。エージェント・ナレッジが増えても、フローの全体像は常に 1 ファイルを読めば分かる。
>
> ステータス: 🚧 draft。実装（Phase 1〜）の知見で随時改訂する。

## 1. 全体構造

```yaml
flow:
  id: hello_survey            # 一意ID（ログ・state ファイル名に使用）
  name: サンプル調査フロー      # 表示名
  version: 1                  # schema バージョン

paths:                        # パス変数（ファイル内で ${VAR} 参照）
  SRC: ./target-repo
  OUT: ./flowsmith-out        # OUT は予約変数・必須（成果物と実行時ファイルの置き場）

defaults:                     # 全ステップ共通のデフォルト
  max_rounds: 3               # make-check の再試行上限

steps:
  - id: ...                   # ステップ定義（後述）
```

トップレベルは `flow` / `paths` / `defaults` / `steps` の 4 ブロックのみ。

## 2. ステップ共通フィールド

```yaml
  - id: inventory             # 一意ID（snake_case）
    type: make                # make | make-check | branch | loop
    title: 技術スタックの棚卸し  # 表示名
    brief: |                  # 作業指示（インライン記述）
      対象リポジトリのビルド定義から技術スタックを特定し、
      EOL 状況とあわせて表にまとめる。
    brief_file: briefs/inventory.md   # ↑が長い場合は外部ファイル参照（どちらか一方）
    refs:                     # 参照資料（任意）: ナレッジ・テンプレート等
      - refs/java-legacy.md
    inputs:
      - id: source
        path: ${SRC}/
        note: 調査対象のルート
    outputs:
      - id: inventory
        path: ${OUT}/inventory.md
    gates:                    # 機械ゲート（LLM を通さない決定的検査）
      - exists: ${OUT}/inventory.md
      - sections: ["サマリー", "未確認事項"]   # 必須見出し
    next: structure           # 無条件ルーティング（省略時は配列の次へ）
```

### gates（機械ゲート）

成果物に対する決定的検査。**ひとつでも落ちたら LLM レビュー以前に差し戻す。**

| キー | 検査内容 |
|------|----------|
| `exists: <path>` | ファイルが存在する |
| `sections: [...]` | Markdown に指定見出しがすべて在る |
| `min_lines: <n>` | 出力ファイルの最低行数 |
| `table_rows_min: <n>` | 出力内の Markdown 表のデータ行数が n 以上 |
| `count_match: {ref: <path>, tolerance: <n>}` | 出力の表のデータ行数が、参照ファイルの表の行数と一致（±tolerance） |

## 3. ステップタイプ別フィールド

### 3.1 `make` — 生産のみ

共通フィールドのみ。レビューなしで成果物を出す。

### 3.2 `make-check` — 生産 + 検査ループ

```yaml
  - id: write_specs
    type: make-check
    title: 仕様書作成
    maker:
      brief: |
        クラスごとの仕様書を作成する。
      refs: [refs/spec-format.md]
    checker:
      brief: |
        仕様書を観点表に基づき検査する。
      viewpoints:             # 検査観点（構造化）
        - id: V01
          severity: high      # high | mid | low
          rule: 全 public メソッドが記載されている
        - id: V02
          severity: mid
          rule: 引数・戻り値の型が原典コードと一致する
    pass_when: no_high_fail   # no_high_fail | all_pass
    max_rounds: 3             # 超過時はエンジンが SEIZED（人間裁定待ち）
```

実行順は **maker → gates（機械）→ checker（LLM）→ 不合格なら指摘を渡して maker 再実行**。
`max_rounds` 超過は次ステップへ逃さず、フロー全体を停止して「行き詰まりサマリー」を出す。

**合否はエンジンが機械的に決める**：checker は観点ごとの判定（verdicts）を結果ブロックで構造化して返し、エンジンが `pass_when` を適用する。checker の自己申告の合否は使わない。レビュー表は `${OUT}/reviews/{step_id}_r{n}.md` に残る。

### 3.3 `branch` — 二者択一の判定と分岐

```yaml
  - id: quality_gate
    type: branch
    title: 調査品質の判定
    arbiter:
      brief: |
        中間成果物に矛盾・根拠なし数値がないか判定する。
    uses: [inventory, structure]     # 判定材料にする上流ステップの結果
    route:
      yes: final_report
      no: structure                  # 戻りジャンプ = ループ構成も可
    max_returns: 3                   # 戻りジャンプの上限（省略時 3）。超過 = seized（無限ループ防止）
    carry:                           # 戻り先に判定理由を引き継ぐ
      to: structure
      keys: [reason]
```

### 3.4 `loop` — 条件付き反復

```yaml
  - id: refine
    type: loop
    title: 改善反復
    arbiter:
      brief: 改善が収束したか判定する。
    route:
      done: next_phase        # 収束 → 抜ける
      again: fix_step         # 未収束 → 反復
    max_laps: 5               # 超過 = SEIZED（人間裁定待ち）
```

## 4. 反復入力（foreach）

表駆動で同じステップを行単位に繰り返す：

```yaml
    foreach:
      table: ${OUT}/class_list.md   # Markdown 表を含むファイル
      key: [パッケージ, クラス名]     # 行を識別する列
    outputs:
      - id: spec
        path: ${OUT}/specs/         # 行キーごとに specs/{key}.md へ
```

state には**行単位の完了状態**が記録され、中断後の再開は未完了行から始まる（完了行は再実行しない）。

## 5. ステップの結果ブロック（機械可読）

各ステップは終了時、出力の末尾に **YAML フェンスの結果ブロック**を出す。エンジンは自然文を解釈せず、このブロックだけを読む：

````markdown
```flowsmith-result
step: write_specs
status: pass          # done | pass | fail | yes | no | again | seized
rounds: 2
outputs:
  - flowsmith-out/specs/
notes: V02 を 2 回指摘の後解消
```
````

| type | 取りうる status |
|------|------------------|
| make | `done` |
| make-check | `pass` / `seized` |
| branch | `yes` / `no` |
| loop | `done` / `again` / `seized` |

ブロックが欠落・不正な場合、エンジンは推測で進まず `seized`（停止・人間待ち）にする。

## 6. 実行時ファイル（概要のみ・詳細は Phase 2 で設計）

```text
flowsmith-out/
├── <成果物>                  # outputs で宣言した場所
└── .flowsmith/
    ├── state.json            # 断点：完了ステップ / 次ステップ / 行単位状態
    ├── run.jsonl             # 流水ログ（start/stop/route/seized、所要時間・トークン概算）
    └── runs/                 # 完了したフローの state アーカイブ
```

同一コマンド再実行で `state.json` を検出し、続きから自動再開する。

---

## 付録: 紙上検証 — レガシー現状調査フローを v0 schema で表現

[legacy-migration-skills](https://github.com/KIKAIKAYA/legacy-migration-skills) の 5 フェーズが書けることの確認：

```yaml
flow:
  id: legacy_survey
  name: レガシー現状調査
  version: 1

paths:
  SRC: ./target-repo
  OUT: ./survey-output

defaults:
  max_rounds: 3

steps:
  - id: inventory
    type: make-check
    title: 技術スタック棚卸し
    maker: { brief_file: briefs/inventory.md }
    checker:
      viewpoints:
        - { id: V01, severity: high, rule: 全項目に根拠ファイルが併記されている }
        - { id: V02, severity: high, rule: 数値はコマンド実行結果に基づく }
    gates:
      - exists: ${OUT}/01-inventory.md
      - sections: ["サマリー", "未確認事項"]
    pass_when: no_high_fail

  - id: structure
    type: make-check
    title: 構造・依存の可視化
    maker:
      brief_file: briefs/structure.md
      refs: [${OUT}/01-inventory.md]
    checker:
      viewpoints:
        - { id: V11, severity: high, rule: エントリポイントが種類別に全数列挙されている }
    gates:
      - exists: ${OUT}/02-structure.md

  - id: database
    type: make-check
    title: データ層調査・移行評価
    maker:
      brief_file: briefs/database.md
      refs: [${OUT}/01-inventory.md, ${OUT}/02-structure.md]
    gates:
      - exists: ${OUT}/03-database.md

  - id: consistency_gate
    type: branch
    title: 検収ゲート
    arbiter:
      brief: 01〜03 に矛盾・根拠なし数値・確証度未記載がないか判定する。
    uses: [inventory, structure, database]
    route: { yes: report, no: structure }
    carry: { to: structure, keys: [reason] }

  - id: report
    type: make-check
    title: 調査報告書生成
    maker: { brief_file: briefs/report.md }
    checker:
      viewpoints:
        - { id: V31, severity: high, rule: 1章に技術用語が含まれない }
        - { id: V32, severity: high, rule: すべての数値が中間成果物から引用できる }
    gates:
      - exists: ${OUT}/00-report.md
      - sections: ["エグゼクティブサマリー", "リスク一覧", "未確認事項一覧"]
```

✅ 5 フェーズ・検収ゲート・戻りループ・二層報告の検査がすべて表現できる。v0 として成立。
