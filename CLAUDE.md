# 開発方針＆開発環境ルール(RS-TypeScript)

作業ドライブは`F:\runo`。この節は[`open-raid-z`](https://github.com/aon-co-jp/open-raid-z)の`CLAUDE.md`を正本とし、各プロジェクトへコピーして同期する方針に準じる。

## リポジトリ改称(2026-09-13)

`RTypeScript`→`RS-TypeScript`へGitHub上でrename済み(crate名も
`rtypescript`→`rs-typescript`)。`aruaru.pro`(コンコナラ型スキル
マーケットプレイス+求人サイト統合の新規プロジェクト)向けの
フロント基盤整備に合わせた、`RFrontEnd`傘下プロジェクトのネーミング
統一(`RJSON`→`RS-JSON`統合・`RBootstrap`→`RS-BootStrap`・
`RS-JavaScript`新設)の一環。以下の記述内の`RTypeScript`表記は
改称前の履歴として残す。

## このプロジェクトの構想

`RHTML5`/`RCSS3`/`RTypeScript`/`RBootStrap`という4プロジェクト構想の1つ。
詳細な全体構想は[`rhtml5`](https://github.com/aon-co-jp/RTHML)のCLAUDE.mdを参照(構想はrhtml5側に集約して記録)。

**スコープが非常に大きい(フルの型システム・型推論・構文解析)ため、
無理に全体実装を狙わず最小のスコープから立ち上げる方針**(ユーザー
指示、2026-07-18)。第一段は「単純な型注釈の削除・JSへの
トランスパイルのみ」に絞り、フル型チェックは対象外とする。

以前に検討された3つの実装方針案(A: 独自VM実行、B: Rustネイティブ
DSL+Wasm直接コンパイル、C: swcでASTのみ取り込み+独自インタプリタ)は
将来のフル実装フェーズでの選択肢として維持するが、第一段の
トークナイザ+型注釈除去はいずれの案にも先立つ土台であり、方針決定を
待たずに着手した。

## 現状(第一段、2026-07-18)

- `src/token.rs`: `Token`列挙型(`Identifier`/`Number`/`StringLiteral`/
  `Punctuator`/`Whitespace`/`Comment`/`Eof`)、`TokenSink`トレイト、
  テスト用`CollectingSink`。`rhtml5::token`と同じ疎結合パターンを踏襲
  (異なる点: 空白・コメントも独立トークンとして保持し、トランスパイル
  時にフォーマットを保存できるようにしている)。
- `src/tokenizer.rs`: 識別子・数値・文字列リテラル(`"`/`'`/`` ` ``、
  素朴なバックスラッシュエスケープ対応)・行/ブロックコメント・
  記号(多文字演算子は最長一致)・空白の字句解析器。
- `src/transpile.rs`: トークン列から型注釈を除去してJS相当の文字列を
  再構成する`transpile`/`strip_type_annotations`。対応: 変数宣言
  (`let`/`const`/`var`直後の識別子の`:`)、関数パラメータ(`(`/`,`直後の
  識別子の`:`)、関数の戻り値型注釈(`)`直後の`:`、終端が`{`または`=>`
  であることを確認してから確定するフォールバック付き)。型注釈の終端は
  `(`/`[`/`{`のネスト深さ追跡で判定(配列型・関数型のような入れ子を
  誤って途中で打ち切らないため)。オブジェクトリテラルの`key: value`・
  三項演算子の`cond ? a : b`は型注釈の文脈に一致しないため保持される
  ことをテストで確認済み。
- **未対応(次段階)**: `interface`/`type`宣言、ジェネリクス制約、複数
  変数宣言子の2つ目以降(`let x: T = 1, y: U = 2;`のyの型は残ってしまう)、
  Wasmへの直接コンパイル、swc AST取り込み。
- **検証**: `cargo test`で14件全green(トークナイザ5件+トランスパイル
  9件、単純な変数宣言・関数パラメータ・戻り値型・アロー関数・配列型/
  関数型・非型注釈コロンの非破壊・空白コメント保存を含む)。警告0件。

## 次にすべきこと

1. 複数変数宣言子対応(`let x: T = 1, y: U = 2;`)
2. `interface`/`type`宣言の扱い(第一段では無視して素通りさせている
   だけなので、パース時にエラーにするか読み飛ばすか方針を決める)
3. 実装方針の最終決定(B案「Rustネイティブdsl+直接Wasmコンパイル」
   から着手しC案「swc AST+独自インタプリタ」へ拡張、が暫定方針)

## 関連プロジェクト

- [rhtml5](https://github.com/aon-co-jp/RTHML) / [rcss3](https://github.com/aon-co-jp/RCSS) / [rreact](https://github.com/aon-co-jp/RReact) — 全体構想の詳細、相互統合の現状はこちら側に記録
- [open-raid-z](https://github.com/aon-co-jp/open-raid-z) — 開発ルールの正本

## HANDOFF

- **2026-07-19 audiocafe-tokyo-rustユーザーからの完成度向上要望を記録
  (次回セッションの着手事項)**: `audiocafe-tokyo-rust`のユーザーから
  「未着手や未完成の技術があれば、それぞれのリポジトリもTESTしながら
  完成させていって、実用性と完成度を高めていって下さい」という要望が
  あった。各リポジトリを別セッションで順に実施する方針となったため、
  本セッションでは実装は行わず、次回セッションでは上記「次にすべき
  こと」1〜3(複数変数宣言子対応・`interface`/`type`宣言の扱い・
  実装方針B/C案の最終決定)へ確認不要でそのまま着手すること。
  現状(2026-07-18時点)は最小スコープの型注釈除去トランスパイラの
  みで、実際にどこかのプロジェクトから使われている段階ではない
  (`audiocafe-tokyo-rust`はクライアント側JSをほぼ持たない設計のため
  現時点で不要)。

- **2026-07-18 リポジトリ新設・最小スコープでの立ち上げ**: 完全に
  空だったリポジトリに、`Cargo.toml`と`token`/`tokenizer`/`transpile`
  の3モジュールを新規実装。フル型システムを狙わず「単純な型注釈の
  削除・JSへのトランスパイルのみ」というユーザー指示のスコープに
  絞った。`cargo test`で14件全green、警告0件を確認。`git init`済み、
  GitHub側`aon-co-jp/RTypeScript`リポジトリは既に存在していたため
  originとして紐付けてpush予定。
  次にすべきこと: 複数変数宣言子対応、`interface`/`type`宣言の扱いの
  方針決定、Wasmコンパイル方針(B/C案)の最終決定。

## API 方針: REST 全廃・gRPC / Connect へ (2026-11-04)

### 日本語
- **REST は全廃する。新しい API は REST で作らない。** ブラウザや `curl` から呼ぶ API は **Connect の単純呼び出し**(`POST /<パッケージ>.<Service>/<Method>`、`content-type: application/json`、本文は JSON)、サービス間は **gRPC / gRPC-web**(RPoem の管理API `open_runo.admin.v1.Admin` と同じ流儀)。ストリーミングは、必要になってから。
- **既存の REST は、触るときに順に移す。** 移したら旧 REST の経路は消す。移行が終わるまで残るが、**新規には追加しない**。
- **JSON は RS-JSON に統一**する(読み込みは `from_str_secure` / `parse_secure` でサイズ・深さ・重複キーの上限つき、書き出しは `rust_json::fast`)。ブラウザの WASM に `prost` は入れない(サイズが増えるため)。
- **例外(自分たちの API ではなく、相手の規格が決めているもの)**: ACME、OAuth/OIDC のコールバック、第三者(GitHub など)からの Webhook、ヘルスチェック、静的ファイル配信、OTLP など。これらは規格どおりの HTTP を受ける。
- 参考実装: `RPoem`(`open_runo.admin.v1.Admin`)、`open-easy-web`(`server/src/connect.rs`、`proto/open_easy_web/admin/v1/admin.proto`)。
- ※ このリポジトリのコードが既に方針どおりかどうかは、各リポジトリの現状による。この節は方針の記録であり、コードの変更を意味しない。

### English
- **REST is being abolished. New APIs are not built as REST.** APIs called from browsers or `curl` use **Connect unary calls** (`POST /<package>.<Service>/<Method>`, `content-type: application/json`, JSON body); service-to-service calls use **gRPC / gRPC-web** (the same style as RPoem's admin API `open_runo.admin.v1.Admin`). Streaming only when it is actually needed.
- **Existing REST endpoints are migrated one by one, when they are touched.** The old REST route is removed once migrated. Until migration is finished they remain, but **no new ones are added**.
- **JSON is unified on RS-JSON**: reading with `from_str_secure` / `parse_secure` (size, depth and duplicate-key limits), writing with `rust_json::fast`. `prost` is not put into browser WASM (it would grow the binary).
- **Exceptions (shapes dictated by someone else's standard, not our own API)**: ACME, OAuth/OIDC callbacks, webhooks from third parties (e.g. GitHub), health checks, static file serving, OTLP, and so on. These accept plain HTTP as the standard requires.
- Reference implementations: `RPoem` (`open_runo.admin.v1.Admin`) and `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Note: whether the code in a given repository already follows this policy depends on that repository. This section records the policy; it does not mean the code has been changed.

### Deutsch (KI-Übersetzung, noch nicht von Muttersprachlern geprüft)
- **REST wird abgeschafft. Neue APIs werden nicht als REST gebaut.** APIs, die aus dem Browser oder mit `curl` aufgerufen werden, nutzen **Connect-Unary-Aufrufe** (`POST /<Paket>.<Service>/<Methode>`, `content-type: application/json`, JSON-Body); zwischen Diensten gilt **gRPC / gRPC-web** (wie bei der Admin-API von RPoem, `open_runo.admin.v1.Admin`). Streaming erst, wenn es wirklich nötig ist.
- **Bestehende REST-Endpunkte werden einzeln migriert, sobald man sie anfasst.** Nach der Migration wird die alte REST-Route entfernt. Bis dahin bleiben sie bestehen, es werden aber **keine neuen hinzugefügt**.
- **JSON wird auf RS-JSON vereinheitlicht**: Lesen mit `from_str_secure` / `parse_secure` (Grenzen für Größe, Tiefe und doppelte Schlüssel), Schreiben mit `rust_json::fast`. `prost` kommt nicht in Browser-WASM (die Datei würde größer).
- **Ausnahmen (Formate, die ein fremder Standard vorgibt, nicht unsere eigene API)**: ACME, OAuth/OIDC-Callbacks, Webhooks von Dritten (z. B. GitHub), Health-Checks, statische Dateien, OTLP usw. Diese nehmen schlichtes HTTP entgegen, wie es der Standard verlangt.
- Referenzimplementierungen: `RPoem` (`open_runo.admin.v1.Admin`) und `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Hinweis: Ob der Code eines Repositorys diese Richtlinie schon befolgt, hängt vom jeweiligen Repository ab. Dieser Abschnitt hält die Richtlinie fest; er bedeutet nicht, dass der Code bereits geändert wurde.

### Русский (машинный перевод, носителями языка ещё не проверен)
- **REST полностью отменяется. Новые API не создаются как REST.** API, вызываемые из браузера или через `curl`, используют **унарные вызовы Connect** (`POST /<пакет>.<Service>/<Method>`, `content-type: application/json`, тело — JSON); между сервисами — **gRPC / gRPC-web** (так же, как административный API RPoem, `open_runo.admin.v1.Admin`). Потоковая передача — только когда она действительно нужна.
- **Существующие REST-эндпоинты переносятся по одному, когда до них доходят руки.** После переноса старый REST-маршрут удаляется. До завершения переноса они остаются, но **новые не добавляются**.
- **JSON унифицируется на RS-JSON**: чтение через `from_str_secure` / `parse_secure` (ограничения на размер, глубину и повторяющиеся ключи), запись через `rust_json::fast`. `prost` не добавляется в браузерный WASM (размер вырос бы).
- **Исключения (форматы, заданные чужим стандартом, а не наш собственный API)**: ACME, обратные вызовы OAuth/OIDC, вебхуки сторонних сервисов (например, GitHub), проверки работоспособности, раздача статических файлов, OTLP и т. п. Они принимают обычный HTTP, как требует стандарт.
- Эталонные реализации: `RPoem` (`open_runo.admin.v1.Admin`) и `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Примечание: соответствует ли код конкретного репозитория этой политике, зависит от самого репозитория. Этот раздел фиксирует политику и не означает, что код уже изменён.
