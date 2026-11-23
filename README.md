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

## 復号・エミュレーション・文法抽出

以下は検体をホストで起動せず、静的に取得したバイト列だけを扱う補助スクリプトです。

- `scripts/decode_api_names.py`: 符号化された API/DLL 名リテラルの復号と、ROR13-add / CRC32 / djb2 / FNV-1a によるハッシュ候補照合（[API hash解析](malware-re-analysis-ja/references/api-hash-analysis.md)）
- `scripts/offline_emulate.py`: 副作用のないコード範囲を Unicorn 上で API スタブ付き・上限付きでエミュレートし、初期化鍵・設定を復元（[オフライン・エミュレーション](malware-re-analysis-ja/references/offline-emulation.md)）。結果は `OBSERVED_STATIC` であり、検体の起動ではない
- `scripts/request_grammar.py`: 乱数から組み立てられる URI/Cookie の生成文法を出力（[文法抽出](malware-re-analysis-ja/references/request-grammar-analysis.md)）。具体値は `NOT_OBSERVED_RUNTIME` のまま扱う

いずれもネットワークへ接続せず、検体を実行しません。エミュレーションは命令数・時間・ヒープの上限を必須とし、実 API へ到達させません。

Evidence Auditorは単独でも利用できます。

```text
$malware-evidence-auditor-ja <解析レポート> を監査してください。
```

補助スクリプトはリポジトリのルートから `uv run スクリプト.py` で実行します。
環境の準備は [テスト実行](#テスト実行) を参照してください。

構造検査だけをコマンドラインで実行する場合:

```text
uv run malware-evidence-auditor-ja/scripts/validate_evidence.py REPORT.md [--json] [--provenance manifest.json ...]
```

終了コードは ERROR あり=1、読み取り・引数エラー=2、それ以外=0 です。`PASS` は schema・参照検査の通過だけを意味し、解析結果の真実性や埋め込みコードの全言語対応は含意しません。

manifest が名指しする artifact・script file の hash 再計算と段グラフの再導出は、writer を import しない独立スクリプトで行います。

```text
uv run malware-evidence-auditor-ja/scripts/verify_reconstruction_ledger.py manifest.json [--json] [--require-artifacts] [--sample PATH]
```

## 表の仕様

レポート内の表は `malware-evidence-auditor-ja/references/table-contract.md` の仕様に従います。

- 正規Evidence Tableは `Finding | Evidence class | Program | Function/address | Evidence | Confidence` の順序で6列を並べる。`Analysis state` は末尾7列目だけ。
- 技法別表（`stage-ledger`、`decode-blocks`、`image-map`、`module-map`、`ordinal-import`、`module-contract`、`ioc-summary`、`artifact-chain`、`api-hash`）はヘッダ直前に `<!-- table-type: 型名 -->` を宣言します。宣言は検査免除になりません。`Evidence` 列は観測内容、class を置く列は `Evidence class` です。
- レポート冒頭に Program／module ごとの `ADDRESS_CONVENTION` ブロック（`program`、`image_base`、`va`、`rva`）を置きます（`malware-re-analysis-ja/references/address-module-contract.md`）。
- IoC は `ioc-summary` 表で出自（Origin）と使用確度（Usage）を分け、defang して書きます（`malware-re-analysis-ja/references/ioc-reporting.md`）。
- 引用レポートはコードブロックに置き、独立したレポートとして検査する引用には fence 直前に `<!-- block-role: report -->` を置きます。コードブロック内は Markdown / Python / JSON を内容に応じて解析し、Python・JSON の文字列を表として誤検出しません。

設計経緯は `docs/evidence-validation-design.md` にあります。

## 中断からの再開・証拠付きの図

操作・失敗・保留した解釈は最終報告と分けて記録し、再開時に検体・Program・解析設定・保存証拠を照合します。[作業記録](malware-re-analysis-ja/references/analysis-session.md) と [デコンパイル異常時の手順](malware-re-analysis-ja/references/decompiler-recovery.md) に開始条件、代替手順、終了条件があります。

作業記録をスクリプトで照合する場合は、[記録の仕様](malware-evidence-auditor-ja/references/analysis-session-contract.md) に従って `session.json` を作成し、既存のFindingと再構成台帳に対応づけます。

```text
uv run malware-evidence-auditor-ja/scripts/verify_analysis_session.py session.json --evidence-root case/evidence --current current-identity.json --sample sample.bin --report REPORT.md
```

`--mermaid` を追加すると、検査でERRORがなかった場合に、証拠ID・命令位置・条件を付けた制御フロー図（CFG）を標準出力へ出します。静的に導出した遷移は実線、実行時に観測した遷移は太線、未解決の候補は破線で示します。再開できるかどうかの判定は、入力された識別情報・解析設定と保存ファイルの照合に限られます。実際に開かれているProgramとの一致や、命令の解釈の正しさを保証するものではありません。

## テスト実行

テスト環境は [uv](https://docs.astral.sh/uv/) で管理し、Python 3.13 を使用します。
以下の手順で、補助スクリプトとテストに共通の環境を用意します。
コマンドはすべてリポジトリのルートで実行してください。

1. uv が利用できることを確認します。未導入の場合は [公式のインストール手順](https://docs.astral.sh/uv/getting-started/installation/) に従って導入してください。

   ```sh
   uv --version
   ```

2. ロックファイルに従ってテスト環境を作成します。

   ```sh
   uv sync
   ```

   `.python-version` と `pyproject.toml` により Python 3.13 を選択し、`.venv` を作成します。
   Python 3.13 が未導入の場合は uv が自動でダウンロードします。
   `uv.lock` に固定された開発用依存も導入されるため、Unicorn を使うエミュレーションテストも実行できます。

3. テストに使う Python のバージョンを確認します。`Python 3.13.x` と表示されることを確認してください。

   ```sh
   uv run python --version
   ```

4. 両方の Skill のテストを実行します。

   ```sh
   uv run -m unittest discover -s malware-evidence-auditor-ja/tests
   uv run -m unittest discover -s malware-re-analysis-ja/tests
   ```

   両方とも `OK` と表示されることを確認してください。
   動作確認時は Python 3.13.7 / Unicorn 2.1.4 で、監査側153件・解析側56件の計209件がスキップなしで成功しました。

以降も `uv run` で実行すると、`.venv` 内の Python と依存関係が使用されます。
仮想環境を手動で activate する必要はありません。リポジトリ更新後は、次のコマンドで依存関係を同期してからテストを実行してください。

```sh
uv sync
```

検体固有のIoCはtestsやSkillに取り込まず、手作りの最小fixtureだけを使います。

探索の動作評価には [評価入力](malware-re-analysis-ja/evals/exploratory-ioc/cases.md) を解析担当へ渡し、[採点基準](malware-re-analysis-ja/evals/exploratory-ioc/rubric.md) は評価側だけが読みます。完成済み IoC 表や正解値を入力へ与えず、導出・使用先・不確実性の扱いを確認します。構造テストの成功と、探索の動作評価の結果は分けて記録します。

検体は隔離された解析環境で扱い、Linux/Kali上では実行しないでください。
