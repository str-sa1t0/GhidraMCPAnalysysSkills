# 探索的 IoC 取得の動作評価

2026-11-01、[探索的コード解析](../malware-re-analysis-ja/references/exploratory-code-analysis.md) の更新時に実施した。対象は合成コード断片であり、実検体・live Ghidra の解析結果ではない。

## 評価方法

[評価入力](../malware-re-analysis-ja/evals/exploratory-ioc/cases.md) と Skill・関連参照を、会話履歴を持たない別の解析担当へ渡した。[採点基準](../malware-re-analysis-ja/evals/exploratory-ioc/rubric.md)、正解、過去の結果は渡していない。生成物は各回で別の `/tmp` ディレクトリに保存し、評価側が計算結果・報告・記録を採点した。

元断片の実行・import、検体取得、ネットワーク利用は指示していない。有限の整数・byte 計算を独立実装することだけを許可した。検証可能な演算結果は評価側でも出力 byte を照合し、用途・出自・不足証拠は報告と元入力の対応で判断した。

## 初回4ケース

| ケース | 値の導出と用途の確認 | 発見した問題 |
|---|---|---|
| A: C 疑似コード | 17 byte の host を導出し、mode 条件と名前解決→connect を確認。未使用文字列を除外 | 元検体不明の台帳化の問題が共通して存在 |
| B: 無言語 listing | little endian と network order から正しい IPv4／port と使用先を導出 | 固定即値由来の Origin を CONFIG と誤分類 |
| C: 探索用 Python | 2つの試行を区別し、後半から候補文字列を導出。検体の使用先は UNKNOWN として候補に留めた | 元検体不明の台帳化の問題が共通して存在 |
| D: 未知 seed／slot | 生成式と依存を示し、具体値や通信機能を捏造せず終了 | 値の導出・不確実性の扱いに重大な問題なし |

初回は資料ファイルの hash を `sample_sha256` とする manifest を作成した。資料の hash であることは注記していたが、元検体の識別用 field の意味を維持できていない。構造 validator と artifact verifier はともに PASS だったため、形式・バイト同一性の検査だけではこの問題も B の誤分類も検出できないことを確認した。

これを受け、元検体 SHA256 が未取得なら manifest v2 を保留し、資料・自作計算・出力の実 file の size／SHA256 と導出過程を作業メモへ残すよう明示した。また「再構成した値は CONFIG」という既存文書の読み方を修正し、固定即値・復号設定・ホスト情報・出自不明を区別する判断基準を追加した。

## 再評価と最終版の確認

別担当による A/B の再評価では、正しい導出と、架空の検体 hash を作らず台帳化を保留する動作を確認した。一方、B の Origin は CONFIG のままであり、全面合格とは扱っていない。この評価の進行中にも文書更新があったため、最終版を固定し、別の新規担当に B だけを渡して確認した。

最終版での B は以下を満たした。

- 宛先 `198[.]51[.]100[.]42`、port `8080` を導出。byte 列・逆変換・API 引数を照合。
- Origin は `FIXED`、Usage は `OBSERVED_STATIC`。実到達・通信成功・TCP／C2 の断定をしない。
- 未取得の ImageBase・命令 offset・元検体 hash・sockaddr 末尾を創作しない。
- manifest v2 は保留し、実在する資料・計算・結果の hash と導出過程を記録。
- 構造検査は ERROR 0、WARNING 1（ImageBase 未取得に伴う `ADDRESS_CONVENTION_MISSING`）。規約の捏造で警告を消していない。

## 生成物と再現情報

- 初回: [報告](/tmp/exploratory-ioc-forward-p15eirme/report-ja.md)、[計算記録](/tmp/exploratory-ioc-forward-p15eirme/calculations.json)。この回の manifest は上記の不適合を含む評価結果として保持する。
- A/B 再評価: [報告](/tmp/exploratory-ioc-recheck-jcb56z4l/report.md)、[確認記録](/tmp/exploratory-ioc-recheck-jcb56z4l/checks.md)。B の誤分類が残るため最終正解例にはしない。
- 最終版 B: [報告](/tmp/exploratory-ioc-final-b-3rvwqq2s/report.md)、[作業メモ](/tmp/exploratory-ioc-final-b-3rvwqq2s/work-notes.md)、[構造検査](/tmp/exploratory-ioc-final-b-3rvwqq2s/validation.json)。
- 指示の版: [初回 hash](/tmp/exploratory-ioc-review-y_iinjtn/initial-hashes.json)、[最終版 hash](/tmp/exploratory-ioc-review-y_iinjtn/final-hashes.json)。`/tmp` の生成物は一時的であり、再評価には repository の入力と採点基準を使う。

既存テストは auditor 125件（validator 112、verifier 13）、re-analysis 17件が全件成功。両 Skill の `quick_validate.py`、追加参照リンク、`git diff --check` も通過した。これらの件数に AI の動作評価は加算していない。

## 限界

最終版固定後の独立確認は B の1件だけであり、最終版で4ケースすべてを再実行したとは主張しない。live tool での追加取得、多数関数をまたぐデータフロー、実際の packed 検体や未知形式への一般化は未評価。Skill は AI の探索・報告・監査を指示するもので、validator に汎用の意味解析器を実装した変更ではない。先のコードレビューで指摘した writer／verifier／schema 検査の3件の不具合も、この文書更新では修正していない。
