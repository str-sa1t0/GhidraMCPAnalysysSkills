# MalAnalysysSkills

Windowsマルウェア解析用の日本語Codex Skillsです。使用するSkillディレクトリを `~/.codex/skills/` へ配置してください。

## 基本的な使い方

次を呼び出せばマルウェア解析レポートの出力を行います。

```text
$malware-re-analysis-ja FlareVM上の <対象ファイル> を解析し、最終レポートまで作成してください。
```

復号・静的再構成はmain Skillに含まれます。再構成を行った場合は `scripts/write_reconstruction_manifest.py`（1段ずつ）または `scripts/write_reconstruction_ledger.py --spec`（多段チェーン・module・segmentを一括）がmanifest v2を書き、レポートの `stage-ledger` / `module-map` 表と同じartifact ID・SHA256で対応づけます。`$malware-evidence-auditor-ja` が利用可能なら、最終レポートの証拠監査にも使用します。

コード断片から探索的に IoC を取得する場合も main Skill を使います。Decompiler の疑似コード、逆アセンブリ、探索用 Python、言語未指定の断片から入力・変換・使用先を追い、既存の手順にない生成方法も調べます。元検体や完成した証拠表がなくても、提示範囲から開始します。判断基準は [探索的コード解析](malware-re-analysis-ja/references/exploratory-code-analysis.md) にあります。

```text
$malware-re-analysis-ja このコード断片から生成される宛先や設定値を調べ、導出根拠・使用先・未確定事項を報告してください。
```

探索中の仮説・失敗は作業メモへ、採用した結果は既存の Finding・IoC 表・必要な台帳へ記録します。以下の validator は構造・構文を検査し、コードからの値の導出や用途の判断は解析担当と監査担当が行います。

Evidence Auditorは単独でも利用できます。

```text
$malware-evidence-auditor-ja <解析レポート> を監査してください。
```

構造検査だけをコマンドラインで実行する場合:

```text
python3 malware-evidence-auditor-ja/scripts/validate_evidence.py REPORT.md [--json] [--provenance manifest.json ...]
```

終了コードは ERROR あり=1、読み取り・引数エラー=2、それ以外=0 です。`PASS` は schema・参照検査の通過だけを意味し、解析結果の真実性や埋め込みコードの全言語対応は含意しません。

manifest が名指しする artifact・script file の hash 再計算と段グラフの再導出は、writer を import しない独立スクリプトで行います。

```text
python3 malware-evidence-auditor-ja/scripts/verify_reconstruction_ledger.py manifest.json [--json] [--require-artifacts] [--sample PATH]
```

## 表契約

レポート内の表は `malware-evidence-auditor-ja/references/table-contract.md` の契約に従います。

- 正規Evidence Tableは `Finding | Evidence class | Program | Function/address | Evidence | Confidence` の順序で6列を並べる。`Analysis state` は末尾7列目だけ。
- 技法別表（`stage-ledger`、`decode-blocks`、`image-map`、`module-map`、`ordinal-import`、`module-contract`、`ioc-summary`、`artifact-chain`、`api-hash`）はヘッダ直前に `<!-- table-type: 型名 -->` を宣言します。宣言は検査免除になりません。`Evidence` 列は観測内容、class を置く列は `Evidence class` です。
- レポート冒頭に Program／module ごとの `ADDRESS_CONVENTION` ブロック（`program`、`image_base`、`va`、`rva`）を置きます（`malware-re-analysis-ja/references/address-module-contract.md`）。
- IoC は `ioc-summary` 表で出自（Origin）と使用確度（Usage）を分け、defang して書きます（`malware-re-analysis-ja/references/ioc-reporting.md`）。
- 引用レポートはコードブロックに置き、独立したレポートとして検査する引用には fence 直前に `<!-- block-role: report -->` を置きます。コードブロック内は Markdown / Python / JSON を内容に応じて解析し、Python・JSON の文字列を表として誤検出しません。

設計経緯は `docs/evidence-validation-design.md` にあります。

## テスト

```text
python3 -m unittest discover -s malware-evidence-auditor-ja/tests
python3 -m unittest discover -s malware-re-analysis-ja/tests
```

検体固有のIoCはtestsやSkillに取り込まず、手作りの最小fixtureだけを使います。

探索の動作評価には [評価入力](malware-re-analysis-ja/evals/exploratory-ioc/cases.md) を解析担当へ渡し、[採点基準](malware-re-analysis-ja/evals/exploratory-ioc/rubric.md) は評価側だけが読みます。完成済み IoC 表や正解値を入力へ与えず、導出・使用先・不確実性の扱いを確認します。構造テストの成功と、探索の動作評価の結果は分けて記録します。

検体は隔離された解析環境で扱い、Linux/Kali上では実行しないでください。
