# Windows マルウェア解析・証拠監査スキル

Windows 検体やコード断片を調べ、日本語で報告するための Codex スキルです。既存レポートの監査だけを依頼することもできます。

| やりたいこと | 使うスキル | 得られるもの |
| --- | --- | --- |
| 検体やコード断片を解析する | [malware-re-analysis-ja](malware-re-analysis-ja/SKILL.md) | 確認できた挙動、根拠、未確認事項をまとめた報告 |
| 既存レポートを監査する | [malware-evidence-auditor-ja](malware-evidence-auditor-ja/SKILL.md) | 証拠を超えた断定や参照の不整合、その箇所と修正案 |

## 導入と依頼例

使いたいスキルのディレクトリを `~/.codex/skills/` に配置します。両方を配置すると、解析レポートの仕上げにも監査スキルを使えます。

**検体を解析する。** Ghidra MCP から参照できる対象をファイル名で指定します。検体の実行には明示的な許可と隔離された Windows VM が必要です。デバッガーが使えない場合も、静的に確認できた範囲を報告します。

```text
$malware-re-analysis-ja FlareVM 上の sample.exe を静的解析し、根拠を添えて日本語のレポートを作成してください。
```

**コード断片から侵害指標（IoC）を調べる。** 疑似コードや逆アセンブリを依頼文に貼り付けます。元検体や Ghidra 接続は不要です。

```text
$malware-re-analysis-ja 次のコード断片から通信先の候補を調べてください。値の求め方、コード内の使用箇所、まだ確定できない点を示してください。
```

**既存レポートを監査する。** レポートと、照合に使える証拠を指定します。

```text
$malware-evidence-auditor-ja REPORT.md を監査し、証拠の範囲を超えている記述と修正案を示してください。
```

## レポートの形式を検査する

表の列や証拠への参照はコマンドでも検査できます。以下はリポジトリのルートで実行します。環境は Python 3.13 と [uv](https://docs.astral.sh/uv/getting-started/installation/) を使います。

```sh
uv sync --locked
uv run malware-evidence-auditor-ja/scripts/validate_evidence.py REPORT.md
```

静的再構成の記録がある場合は `--provenance manifest.json` を追加します。複数の記録を渡すときはファイルごとに `--provenance` を付けます。`PASS` は形式と参照の検査結果です。各主張が証拠から導けるかは監査スキルで確認します。

- 再構成したファイルと記録のハッシュを照合する手順は、[監査手順](malware-evidence-auditor-ja/SKILL.md#監査手順)を参照してください。
- レポートの表を作るときは[表の仕様](malware-evidence-auditor-ja/references/table-contract.md)、中断した解析を再開するときは[作業記録の仕様](malware-evidence-auditor-ja/references/analysis-session-contract.md)を参照してください。
- 復号、エミュレーションなどの技法別の手順は[解析スキルの概要・索引](malware-re-analysis-ja/OVERVIEW.md)から参照できます。

## 開発用テスト

上記の環境で両スキルの単体テストを実行できます。

```sh
uv run -m unittest discover -s malware-evidence-auditor-ja/tests
uv run -m unittest discover -s malware-re-analysis-ja/tests
```

コード断片から IoC を導けるかを評価するための合成例と採点方法は、[探索的 IoC 取得の動作評価](docs/exploratory-ioc-evaluation.md)にあります。
