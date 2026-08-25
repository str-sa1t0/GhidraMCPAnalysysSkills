# MalAnalysysSkills

Windowsマルウェア解析用の日本語Codex Skillsです。使用するSkillディレクトリを `~/.codex/skills/` へ配置してください。

## 基本的な使い方

次を呼び出せばマルウェア解析レポートの出力を行います。

```text
$malware-re-analysis-ja FlareVM上の <対象ファイル> を解析し、最終レポートまで作成してください。
```

復号・静的再構成はmain Skillに含まれます。`$malware-evidence-auditor-ja` が利用可能なら、最終レポートの証拠監査にも使用します。

Evidence Auditorは単独でも利用できます。

```text
$malware-evidence-auditor-ja <解析レポート> を監査してください。
```

検体は隔離された解析環境で扱い、Linux/Kali上では実行しないでください。
