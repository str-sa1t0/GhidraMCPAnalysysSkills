# MalAnalysysSkills

Windowsマルウェア解析用の日本語Codex Skillsです。各Skillディレクトリを `~/.codex/skills/` へ配置してください。

## 基本的な使い方

次を呼び出せばマルウェア解析レポートの出力を行います。

```text
$malware-re-analysis-ja FlareVM上の <対象ファイル> を解析し、最終レポートまで作成してください。
```

補助Skillsとして、復号・静的再構成に `$malware-static-reconstruction-ja`、最終レポートの証拠監査に `$malware-evidence-auditor-ja` を使用します。

補助Skillは単独でも利用できますが、レポート出力の過程で実行されます。

```text
$malware-static-reconstruction-ja <抽出byte・変換式> を静的に再構成してください。
$malware-evidence-auditor-ja <解析レポート> を監査してください。
```

検体は隔離された解析環境で扱い、Linux/Kali上では実行しないでください。
