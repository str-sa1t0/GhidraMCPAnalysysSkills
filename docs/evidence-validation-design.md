# Evidence validator の表契約・再帰解析設計

状態: 設計案。2026-11-01 に `malware-evidence-auditor-ja/scripts/validate_evidence.py` と両 Skill の文書・tests へ実装済み。実行時の表契約の正本は `malware-evidence-auditor-ja/references/table-contract.md` であり、本書は設計経緯の記録として保持する。両者が食い違う場合は正本と script を優先し、本書を更新する。

追記（2026-11-01）: 別設計「証拠表契約・再構成台帳・アドレス契約・IoC/監査」の残項目を同日に実装した。`ioc-summary` は Origin（`HOST_DERIVED` 表記、`EXTERNAL` / `UNKNOWN` 追加）と Usage の二軸、`artifact-chain` の class 列は `Evidence class`、`module-map` / `decode-blocks` 型、`ADDRESS_CONVENTION` ブロック、defang 検査、独立再検証 `verify_reconstruction_ledger.py` を追加した。本書の第5節にある旧列名・旧表記は正本（table-contract.md）が優先する。

対象は `malware-evidence-auditor-ja/scripts/validate_evidence.py` と、その契約を使う両 Skill の文書・tests。最初に表の識別と再帰解析を実装し、同じ枠組みに技法別検査を追加する。本書の追加だけでは、既存 Skill の規則・CLI・検査結果を変更しない。

## 1. 成立させること

- コードブロックを一律に読み飛ばさず、内容に応じて解析する。
- 本文・埋め込み文書を別の検査範囲にし、表数・ID・参照を混同しない。
- 表の種類の宣言は、検査する schema を選ぶ。検査免除には使わない。
- バイト同一性、命令の存在、意味解釈、実行観測を分離する。
- 不明・未対応・解析打ち切りを記録し、全体を検査したように報告しない。

入力はユーザーが指定した UTF-8 Markdown レポート。外部リンク、コードの import、記載パスを自動的に取得・実行しない。今回の validator は構造と提示済みデータの整合性を検査する。検体の解析、意味的監査、再構成 script の実行は別の責務である。

探索的 IoC 取得の責務は re-analysis の [探索的コード解析](../malware-re-analysis-ja/references/exploratory-code-analysis.md) に定義する。未定型のコード断片から値と使用経路を追うのは解析担当、提示済み導出を照合するのは監査担当である。本書の Python／JSON の構文解析や coverage は、その探索の実施・成功を示さない。採用結果を本書の既存表・provenance へ接続し、探索途中の仮説・失敗・不足証拠は作業メモに保持する。

## 2. 文書と検査範囲

### 2.1 Scope

入力全体を `root` とする。埋め込み Markdown を解析するたびに子 scope を作る。

```text
root                          本文レポート
  block-1                     Markdown 断片
    block-1                   断片内の埋め込みレポート
  block-2                     Python ソースの解析記録
```

scope ID は `root/block-1/block-1` の形式で、その親にあるコードブロックの出現順から決める。Markdown として再解析しないコードブロックにも ID を与える。ID は今回の入力内の診断位置であり、レポート改訂をまたぐ永続 ID には使わない。

各 scope は、親、種別、言語、元ファイル範囲、行対応表、直接所属する表、Finding ID、artifact ID、診断、解析 coverage を持つ。親の走査時に子の本文をもう一度検査しない。

### 2.2 コードブロックの役割

Markdown のコードブロックは、宣言なしなら `fragment`。完全なレポートとして検査するときだけ、開始 fence 直前の最後の非空行に宣言する。

```html
<!-- block-role: report -->
```

許可する役割は `report` と `fragment` の閉集合。`root` は常に `report`。未知・重複宣言は ERROR、対象ブロックが続かない宣言は WARNING。宣言の有効範囲は同じ scope・同じ Markdown container 内だけとする。

| 役割 | 自分の正規 Evidence Table | 内部の表検査 | 親の表数への加算 | 問題の扱い |
|---|---|---|---|---|
| report | 1つ以上必要 | 実施 | しない | 全体診断へ含める |
| fragment | 存在は要求しない | 見つかった表を検査 | しない | 全体診断へ含める |

サンプル表の掲載は fragment で表現する。fragment でも不正な class や列数を許可しない。意図的な不正入力は tests の fixture として扱い、レポートの検査免除機能は追加しない。

`block-role` は内容の言語を変更しない。`report` を Python ブロックに指定しても Markdown へ読み替えず ERROR とする。`fragment` 宣言は Markdown または後述の Markdown 自動検出に該当するブロックにだけ使用できる。

### 2.3 参照の境界

- 表の存在要件、Stage ID、artifact ID の整合性は scope ごとに検査する。
- Finding セルの先頭に任意で `F001: ` の形式の ID を付けられる。形式は `F[0-9]{3,}`、同一 scope 内で一意。参照される Finding は ID 必須。
- `Evidence ref` は同じ scope の正規 Evidence Table の Finding ID を指す。複数参照は `<br>` 区切り。空参照・未定義・重複定義は ERROR。
- `Location` は同じ scope 内の明示 anchor を `#section-id` で参照する。複数は `<br>` 区切り。明示 anchor の形式は単独行の `<a id="section-id"></a>` とし、ID は `[A-Za-z][A-Za-z0-9_-]*`。自動生成の見出し slug に依存しない。
- 最初の実装では scope をまたぐ構造化参照を許可しない。Markdown の一般リンクとして掲載することはできるが、`Evidence ref` の代替にはしない。
- 引用した表を本文の証拠に採用する場合は、出典を保持して本文側に明示的に記載する。子 scope の表を自動昇格させない。

## 3. ブロック解析と再帰

### 3.1 fence を先に構造として認識する

表を行単位で先に検索せず、まず fence・引用・リスト・通常本文を区別する。fence の基準は [CommonMark の fenced code blocks](https://spec.commonmark.org/0.31.2/#fenced-code-blocks)。バッククォートとチルダ、開始以上の長さの同種の終了 fence、インデント、引用・リスト内の container を扱う。

通常の Markdown では fence 内はリテラルである。本設計の再帰解析は、取り出したリテラルを明示的に別文書として解析する追加処理であり、親の fence 判定を変更しない。内側の短い fence を含める場合、外側を長い fence にする。

閉じていない fence は、構造上は scope 末尾までブロックとして扱い、レポート契約上の `FENCE_UNCLOSED` ERROR を付ける。残りの内容を親の本文に戻さない。4スペース等のインデントコードも本文から分離し、言語未指定ブロックとして後述の検出に渡す。

引用・リスト内の一般表は元の scope に属する。コードブロック境界だけが子 scope を作る。非対応 container を読み飛ばす実装は完了扱いにしない。

### 3.2 内容別の dispatch

info string の最初の語を言語識別に使い、小文字へ正規化する。

| 言語・内容 | 解析 | Markdown 再帰 |
|---|---|---|
| `md`, `markdown` | Markdown の子 scope として全文を解析 | する |
| 言語なし、`text`, `plaintext` | Markdown 手掛かりを検出。該当時だけ子 Markdown scope、それ以外は text として記録 | 条件付き |
| `python`, `py` | `ast.parse` による構文解析。ソースを実行しない | しない |
| `json` | JSON として構文解析。重複 key も検出する | しない |
| その他 | 言語・範囲を coverage に記録 | 自動的にはしない |

Markdown 手掛かりは、構造として有効な `table-type` / `block-role` 宣言、表候補、またはコード fence。表候補はヘッダと区切り候補の組で、Evidence class または §4.3 の証拠風ヘッダを含むもの。手掛かりの検出時点では診断を発行せず、子 scope で一度だけ検査する。

明示された言語は内容の推測で上書きしない。Python の文字列や JSON の文字列値に `| Finding |` があっても表にしない。これらの文字列に埋め込まれた文書の抽出は初期実装の対象外で、coverage の限界に記録する。`block-role: report` の言語未指定・text ブロックは Markdown として検査し、表がなければ存在要件で ERROR とする。

Python は [ast.parse](https://docs.python.org/3/library/ast.html#ast.parse) だけを使い、import・eval・exec・生成コードの実行をしない。Python と JSON の構文不成立は、断片である可能性があるため `CODE_SYNTAX_INCOMPLETE` WARNING。構文が通っても、コードの正当性・変換式の正しさ・実行を確認したことにはしない。JSON の重複 key は `JSON_DUPLICATE_KEY` WARNING として保持する。manifest schema の検査は別途明示的に接続し、JSON であるだけで manifest と推定しない。

未対応言語は coverage 上の `unsupported`。無警告の coverage 記録と schema の PASS をもって「全コード解析済み」と表現しない。

既存の禁止別名検査も scope の構造に従う。Markdown の直接所属する文章・セルを対象にし、子ブロックを親から再走査しない。Python の文字列に `OBSERVED_DYNAMIC` が含まれているだけでは Evidence class の誤用にしない。

### 3.3 再帰の停止

初期既定値は、子 scope 深さ16、全コードブロック数1024、全 Markdown scope に投入した UTF-8 byte 数の合計8 MiB。同じ元バイトを再帰解析する場合も合計へ数える。上限以内でも AST 等の parser が資源上限に達した場合は打ち切りを記録する。

上限到達は `PARSE_LIMIT_EXCEEDED` ERROR。該当部分を `truncated` とし、ほかの独立範囲は可能なら検査を継続する。全体を PASS にしない。深さ・ブロック数・投入 byte 数は coverage と共に出力し、停止位置を元ファイルへ対応づける。

## 4. 表の識別契約

### 4.1 宣言の結合

```html
<!-- table-type: canonical-evidence -->
```

ヘッダ直前の最後の非空行にある完全な単独コメントを採用する。空行は許すが、見出し・文章・別コメント・container 境界を越えない。型名は小文字の登録名と完全一致させる。

連続する宣言行は、空行を挟んでも1つの宣言群として数え、同じ型の重複も ERROR。未知 type は、後続表の有無にかかわらず ERROR。孤立宣言は WARNING。未知かつ孤立なら両診断を残す。解釈不能な予約コメント（例: type が空）は `DECLARATION_INVALID` ERROR。最後の宣言だけで先行宣言を隠さない。

`table-type` を fence の直前に置いても、その内部の表には適用しない。必要な宣言を子 Markdown 本文へ記載する。逆に、子にある宣言は親へ漏れない。

### 4.2 識別順序

1. 有効位置の宣言があれば、宣言型を選択する。
2. 宣言がなくヘッダに `Evidence class` があれば `canonical-evidence`。
3. 宣言がなく証拠風ヘッダなら `UNTYPED_EVIDENCE_TABLE` WARNING。
4. それ以外は提示用の表。schema 検査の対象外。

宣言型とヘッダ構造が矛盾すれば ERROR。各型は許可列の閉集合とし、必須列欠落・未知列・重複列・列数不一致・不正値を検査する。例えば canonical の6列を持つ表に artifact-chain と宣言しても、artifact-chain の不足列・未知列で ERROR になる。

正規 Evidence Table の存在数は `canonical-evidence` の表だけを数える。宣言した未知型や技法別表は数えない。不正な canonical 表も canonical の候補として数え、schema ERROR を残す。「valid な正規表がない」と同じ原因の存在エラーを重ねない。

### 4.3 証拠風ヘッダ

表の構造があることを前提に、正規化したヘッダが次のいずれかに該当する場合とする。

- `Finding` を含み、`Evidence` または `Confidence` を含む。
- `Evidence` と `Confidence` の両方を含む（既存 Artifact chain を含む）。
- 登録済み技法型の識別組を含む: Stage ID + Transform、Image + Segment + VA start、DLL + Ordinal、Module ID + Selector、Value + Origin + Evidence ref。

これは WARNING 用の検出条件であり、技法型の自動選択には使わない。ヘッダ比較は trim、連続空白の縮約、大小文字の同一視、表示用バッククォート・強調の除去まで。slash や hyphen を無差別に消して異なる列名を同一化しない。

### 4.4 行構造

区切り候補は各セルが `:?-{3,}:?` に一致する行。ヘッダ・区切り・データ行のセル数を別々に比較する。列数が不一致でも候補を捨てず、型を識別して `TABLE_SEPARATOR_WIDTH` / `TABLE_ROW_WIDTH` ERROR を出す。証拠風の未宣言表も構造検査は行う。提示用の表は §4.2 のとおり検査対象外。

escape 済み pipe と inline code 内の pipe の分離抑止は、現行 tests が要求する互換動作として保持する。後者は本 validator の寛容な入力形式であり、[GFM tables](https://github.github.com/gfm/#tables-extension-) の完全準拠を主張しない。`<br>`、`<br/>`、`<br />` は ID/hash 等の複数値フィールドで同じ区切りとし、空要素は禁止。数値は decimal または `0x` hex、範囲は半開区間で統一する。

## 5. 型レジストリ

必須列は空欄不可。ただし正規表の任意 Analysis state セルは空欄可。列名・列挙値に別名を追加する場合は契約と tests を同時変更する。`UNKNOWN` は正規 Evidence class の一つであり、hash・数値・参照の代わりには使えない。

| Type | 必須列 | 任意列 | 主な検査 |
|---|---|---|---|
| canonical-evidence | Finding, Evidence class, Program, Function/address, Evidence, Confidence | Analysis state | 指定順6列、state は末尾の7列目だけ、class/state/Confidence の正規値 |
| stage-ledger | Stage ID, Input artifact ID, Input SHA256, Transform, Basis, Output artifact ID, Output SHA256, Evidence class | Input range, Key/seed source, Implementation, Output size | 複数値の対応、hash、段・artifact の接続 |
| image-map | Image, Segment, File start, File size, VA start, Mem size | Zero fill, Overlap, Alignment, Relocation, VA bias | ファイル領域と仮想領域の区別、範囲・重複・換算 |
| ordinal-import | DLL, Ordinal, Resolved name, Source, Role, Evidence class, Confidence | Version, Architecture, Call-site, Evidence ref | ordinal 整数、出典非空、class/Confidence |
| module-contract | Module ID, Selector, Slot, Evidence class, Confidence | Module name, Slot table, Initializer, Entry, Teardown, Evidence ref | Module ID + Selector + Slot の複合キー、列挙値 |
| ioc-summary | Value, Category, Origin, Usage/conditions, Evidence ref, Location, Confidence | Generation rule, Inputs | Origin/Confidence、参照・anchor の実在、host-derived の条件 |
| artifact-chain | Stage, Mechanism, Evidence, Confidence | Remote/Source, Artifact, Expected Local Path | この型の Evidence 列は class 5値、Confidence 3値 |

canonical 以外の列順は自由。正規値は現行の class 5値、state 4値、Confidence 3値を維持する。artifact-chain の Evidence は既存表との互換のための例外で、通常の Evidence 欄の説明文とは異なることを明記する。

### 5.1 stage-ledger と provenance

ledger は入出力の確定した段だけを記録する。未完了段は未解決事項として入力ID・不足情報・中断理由を記録し、後続段の確定入力として接続しない。

Stage ID は scope 内で一意。ID と SHA256 は `<br>` の要素順で対応し、入力側・出力側それぞれ件数一致が必須。SHA256 は64桁 hex、比較前に小文字へ正規化する。出力IDは1つの生成段だけを持つ。同じIDへの異なるhash登録は ERROR、異なるIDの同一hashは許可する。Basis は同一 scope の Finding ID を参照し、元コードとの対応づけを記録する。

入口 artifact と最終 artifact ID は manifest v2 の `sources[]` / `final_artifact_ids[]` で明示する。`stages[]` は `inputs[]` / `outputs[]`、transform、implementation、basis、evidence_class を持つ。各 artifact ID は固定した全バイト列を表し、SHA256/size はその全体に対する値。`range` と ledger の Input range は、そのバイト列を取得した元領域の座標を記録する provenance であり、参照時に暗黙に slice を選ぶ指定にはしない。座標は source ID、address space（file-offset / VA / blob-offset）、start、length を持つ。

既存 artifact の一部だけを次段へ渡す場合は、明示的な extract 段で新しい artifact ID と hash を作る。元 artifact と切り出した artifact を同一IDにしない。これにより、前段出力と次段入力は同じ全バイト列同士で比較できる。range の長さと取得バイト数を照合し、slice hash と元 artifact 全体の hash を混同しない。ledger の複数 Input range / Output size を記載する場合も、対応するIDと件数を一致させる。

`--provenance PATH` を追加し、ユーザーが明示した manifest だけを読み込む。Markdown に書かれたパスから自動取得しない。scope との対応は manifest の `report_scope`（root なら `root`）で固定し、対応不一致は ERROR。ledger の ID/hash を manifest と照合する。複数 scope を扱う場合は引数を繰り返す。

全出力IDを先に収集した後、段を順に検査する。入力は登録済み sources または前段出力だけを許可し、後段出力への参照は ERROR。最終IDの存在と入口からの到達性を検査する。最終出力に使われない探索段は、それだけで ERROR にしない。

manifest 未指定でも ledger 内部のhash矛盾・前方参照・重複生成は検出する。ただし未生成の入力を正当な入口とみなさず、入口検証と最終到達性は `not_checked`、`PROVENANCE_REQUIRED` WARNING とする。artifact ファイルのhash再計算は独立再検証に属し、この照合だけで行ったことにしない。

hash が支えるのはバイト同一性であり、変換式の正しさではない。変換根拠の意味的な妥当性は auditor が確認する。

### 5.2 image-map / module-contract

image-map の1行は1 segment。File size はファイルに存在する長さ、Mem size は仮想領域長とし `0 <= File size <= Mem size`。両領域は start と size の半開区間で表す。ファイル裏付け領域内だけで次を使う。

```text
VA = VA start + (file offset - File start)
```

後続の zero-fill に file offset を捏造しない。Zero fill を記載する場合は末尾の補充長を数値で記録し、`Mem size - File size` と照合する。仮想領域の重複は各該当行に Overlap の説明がなければ ERROR、説明があれば意味的監査へ渡す。Alignment は記載時に正の整数で開始VAとの整合を検査する。VA bias は定義を確認した場合だけ記載する。

ファイル総サイズに対する上限検査は、明示された同一 Image ID の artifact size がある場合だけ可能。なければ `IMAGE_SIZE_UNVERIFIED` WARNING とし、表だけから全範囲を確認できたとしない。Relocation の適用、重複の優先順位、独自 flatten 規則の正しさは数値整合性だけでは証明しない。

module-contract の1行は1つの selector/slot 対応。Module ID 単体の一意性は要求しない。共通 initializer/teardown の複数行への出現を許可する。未解決 slot は `UNKNOWN` と記載できるが、その行から能力名を確定しない。同じ複合キーの重複は ERROR。複合キーが同じで候補が競合するときは、確定表の行を増やさず未解決事項へ記録する。

### 5.3 ioc-summary

Origin は `FIXED` / `CONFIG` / `HOST-DERIVED`。Confidence は用途解釈の確信度。host-derived は Generation rule と Inputs の両方を必須にし、Value に生成規則を示す記号名を置く。未知の生成済み具体値を捏造しない。

1つの ioc-summary 表に HOST-DERIVED と FIXED/CONFIG を混在させない。同じ型の別表を設け、具体値一覧と生成規則を分ける。参照が存在しても、本文がその用途を支持するか、接続・実行を含意しないかは意味的監査に残す。

## 6. 診断・coverage・終了状態

既存 JSON の `file` / `errors` / `warnings` / `findings` と、Finding の `severity` / `line` / `message` は保持する。診断に次を追加する。

```json
{
  "severity": "ERROR",
  "line": 120,
  "end_line": 120,
  "code": "EVIDENCE_CLASS_INVALID",
  "scope": "root/block-2/block-1",
  "local_line": 8,
  "table_type": "canonical-evidence",
  "message": "Invalid Evidence class: OBSERVED_DYNAMIC"
}
```

`line` は常に元ファイルの1始まりの行。子本文のローカル行番号から、引用・リストの prefix 除去も含む行対応表を合成して算出する。全体の問題は root の1行目、表の問題はヘッダ・区切り・該当行、未閉鎖 fence は開始行へ付ける。

診断の重複排除キーは scope + code + 元範囲 + 対象列。順序は元行、scope、code、message で固定する。同じ行の別問題は消さない。子の ERROR は root の canonical 不足 ERROR を抑止しない。

coverage はブロックごとに `parsed-markdown` / `parsed-python` / `parsed-json` / `text-only` / `unsupported` / `truncated` を記録し、診断件数と別に出力する。レポート内のコードが全言語対応であることは PASS の意味に含めない。

CLI は ERROR があれば1、読み取り・引数エラーは2、それ以外0を維持。WARNING があれば表示 verdict は `PASS_WITH_WARNINGS`。ERROR/WARNING がなくても表示は「schema・参照検査 PASS」に限定し、意味的正当性や解析 coverage の完全性を含意しない。

## 7. 実装の分担

依存順は次とし、正規表の行検索に fence 除外条件を後付けする構造にはしない。

```text
source + line map
  -> Markdown block parser
  -> scope / language dispatch / recursion
  -> declaration binding + raw table candidates
  -> table registry + local schema validation
  -> scope-local symbols + cross-table/provenance checks
  -> diagnostics + coverage + CLI result
```

最初は現行 script 内の関数・データ構造で分離し、独立した利用実態のない共通 framework は追加しない。parser 選定では、raw header/区切り行の取得、container、未閉鎖 fence、source map の受け渡しを必須条件にする。レンダリング済み HTML だけから検査しない。

実行時の表契約の正本は auditor の `references/table-contract.md` に置く。実装時に本書の確定部分を移し、両方を独立に更新する正本にはしない。re-analysis は単独配置に必要な最小契約を保持し、他 Skill へのリンクだけに依存しない。リポジトリの tests で両 Skill の見本ヘッダ・列挙値を照合する。

## 8. 例: 引用レポートを再帰解析する

次の外側5連 fence は本書での表示用。実際の入力はその内側である。

`````markdown
# 本文レポート

<!-- block-role: report -->
````markdown
# 引用したレポート

<!-- table-type: canonical-evidence -->
| Finding | Evidence class | Program | Function/address | Evidence | Confidence |
|---|---|---|---|---|---|
| F001: 命令列の存在 | OBSERVED_STATIC | module-image | offset:0x20 | byte 付き listing | HIGH |

```python
example = "| Finding | Evidence class |"
```
````
`````

期待結果:

- `root`: 正規 Evidence Table がないため ERROR。
- `root/block-1`: report。正規 Evidence Table を1つ認識する。
- `root/block-1/block-1`: Python として構文解析する。文字列を表にしない。
- 子の F001 は root の Evidence ref を解決しない。
- 子の class を不正値に変更すれば、元ファイル上の該当行へ追加 ERROR を返す。

## 9. 受け入れテスト

| 領域 | 必須ケース・期待結果 |
|---|---|
| canonical | 既存6列は PASS。state は末尾だけ。中間配置・未知列・重複列は ERROR |
| 宣言 | 型選択、空行許容、文章で結合解除、未知型 ERROR、重複 ERROR、孤立 WARNING |
| 宣言の境界 | 親宣言が子表へ漏れない。子宣言が親へ漏れない。container を越えて結合しない |
| 型による迂回 | 別型宣言でも不足列・未知列・不正class は ERROR |
| 表構造 | 区切り列数・行列数の不一致を ERROR。escape/inline code pipe の既存互換 |
| 未宣言 | 証拠風は WARNING。提示用は無視。正規表がなければ別途 ERROR |
| 存在数 | stage-ledger だけの report は ERROR。子の正規表も親の存在要件を満たさない |
| Markdown 再帰 | md、長短 fence、チルダ、インデント、引用・リスト、2段以上の子を検査 |
| 役割 | report は存在要件を検査。fragment は要件なし。不正な表は両者とも ERROR |
| コード | Python/JSON の文字列内の表を誤検出しない。未指定/text の手掛かりで再帰 |
| 構文・coverage | 未完のコードは WARNING。未対応言語を記録。未閉鎖 fence は ERROR |
| ID・参照 | 同一scopeの参照解決、未定義・重複ERROR。子・兄弟からの暗黙解決を禁止 |
| source map | 多段入れ子・container内の診断が元ファイルの正しい行を指す |
| 停止 | 深さ・件数・byte上限の境界と超過。truncated と ERROR。偽の全体PASSを禁止 |
| stage | 入口登録、前段hash一致、前方参照ERROR、未登録入力ERROR、複数値件数不一致ERROR |
| provenance | 明示入力だけ読む。scope不一致ERROR。未指定は未検査WARNING。sliceと全体hashを混同しない |
| graph | 重複生成ERROR、別ID同hash許容、最終IDの存在・到達性、無関係な探索段の許容 |
| image/module | ファイルとzero-fillの境界、重複説明、サイズ未検証、複合キーの一意性 |
| IoC | Origin、host-derived生成規則・入力必須、同表混在ERROR、Finding/anchorの実在 |
| 回帰 | 実際に利用可能な既存レポートに新旧validatorを適用し、意図した差分を記録 |

既存テストの Analysis state 中間配置は、新仕様の許容ケースから拒否ケースへ移す。旧 validator で PASS した事実だけを、新仕様の互換性確認には使わない。レビューで挙がった既存9レポートは、本作業で読み直していないため回帰結果として扱わない。検体固有IoCを tests や Skill に取り込まず、手作りの最小fixtureを使う。

## 10. 実装順と完了条件

1. block parser・source map・scope・再帰とcoverageを実装する。
2. 宣言と表レジストリを接続し、canonical と既存 artifact-chain を検査する。
3. ID/anchor と技法別の行検査を追加する。
4. manifest v2 と ledger の横断検査を接続する。未接続の間は未検査WARNINGを残す。
5. 両 Skill の文書を同じ契約へ移し、既存レポートの移行差分を確認する。

各段の機能と tests を同時に変更する。未実装の登録型を黙って受理しない。部分実装時に現れる未実装型は `TABLE_VALIDATOR_UNAVAILABLE` ERROR として、coverageにも記録する。最終完了条件は、受け入れテスト、利用可能な実レポート回帰、両 Skill の文書整合、および未検査範囲の明示である。構造検査の完了を、解析結果の真実性の保証にはしない。
