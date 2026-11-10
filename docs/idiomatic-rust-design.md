# Rustらしい（イディオマティックな）設計ポイント

このリポジトリでは、Rustの型システム、所有権モデル、非同期エコシステムを活かした「Rustらしい（イディオマティックな）設計」が随所に採用されています。代表的なポイントを7つ挙げます。

---

## 1. トレイトと関連型（Associated Types）によるゼロコスト抽象化と拡張性

プラグインアーキテクチャの核となる `StorageBackend` トレイトや `UserDetailProvider` トレイトでは、ジェネリクスと関連型（`type Metadata` など）が活用されています。

- `Server<Storage, User>` 自体が `Storage` と `User` の型パラメータを持っており、動的ディスパッチ（`dyn`）が不要な箇所ではコンパイル時に単相化（Monomorphization）される設計です。
- 引数にも `P: AsRef<Path>` のような標準トレイト境界を使うことで、`&str`、`Path`、`PathBuf` などを柔軟に受け取れるようになっています。

## 2. ジェネリック型ネストによる静的ミドルウェアチェーン

コントロールチャネルのイベント処理（`control_loop.rs`）では、`ControlChanMiddleware` トレイトを実装した構造体を段階的にラップ（デコレータパターン）しています。

```rust
let event_chain = PrimaryEventHandler { ... };
let event_chain = EventDispatcherMiddleware::new(..., event_chain);
let event_chain = ActivePassiveEnforcerMiddleware { next: event_chain, ... };
let event_chain = AuthMiddleware { next: event_chain, ... };
// ... (FTPS強制、ロギング、メトリクス)
```

- 各ミドルウェアが内部フィールド `next: Next` をジェネリック型として保持するため、`Box<dyn ControlChanMiddleware>` のようなヒープ割り当てや仮想関数テーブル（vtable）を介さず、単一の複合型としてコンパイル時に解決される（Towerの `Service` に近い）Rustらしい手法です。
- また、変数シャドーイング（`let event_chain = ...`）を使って、パイプラインの組み立てを宣言的かつ安全に記述しています。

## 3. 代数的データ型（Enum）によるプロトコル・メッセージ表現

FTPは状態やコマンドが多いプロトコルですが、文字列や数値のままで引き回さず、早い段階で型安全な `enum` に変換しています。

- **FTPコマンドの表現**: `Command` enum により、各コマンドに必要な引数（例：`Retr { path: String }` や `Pasv`（引数なし））をバリアントごとに厳密に保持します。
- **チャネル間通信メッセージ**: コントロールチャネルとデータチャネル間の非同期通信でも、`DataChanCmd` や `ControlChanMsg` といった `enum` を定義し、`match` 式による網羅性検査（Exhaustiveness checking）の恩恵を受けています。

## 4. Newtype パターンによるセキュリティ・ドメイン保護

単なるプリミティブ型やバイト列をそのまま使わず、意味や制約を持たせるために構造体でラップする **Newtype パターン** が使われています。

- **`Password`**: パスワードのバイト列（`Bytes`）をラップし、`Display` と `Debug` トレイトを手動実装して常に `"*******"` を出力するようにしています。これにより、`Command` 全体を誤ってログ出力（`{:?}`）しても型レベルでパスワード漏洩が防止されます。
- **`TraceId`**: `u64` をラップし、常に16進数フォーマット（`{:#x}`）でログに出力されるよう保証しています。

## 5. アクター風のメッセージパッシングと最小限の共有状態

FTPは1つのセッションにつき「コントロール接続」と「データ接続」の2本のTCPストリームが並行して動きます。

- タスク間の連携には `tokio::sync::mpsc` チャネル（`chancomms.rs`）と `tokio::select!` マクロ（`control_loop.rs`）を使い、コマンド受信・データ転送完了通知・アイドルタイムアウト・シャットダウンシグナルを単一ループ内で競合なく多重化しています。
- セッション状態 `SharedSession` は `Arc<tokio::sync::Mutex<Session<S, U>>>` で共有されますが、ロック保持時間を最小化するため、必要なデータ（`Arc<Storage>` やユーザー情報など）をクローンして即座にロックを解放してから重い非同期I/Oを行う設計になっています。

## 6. Builder パターンと型安全なデフォルト構成

`ServerBuilder` による設定構築が採用されています。

- `ServerBuilder::new(...)` を呼ぶと、認証器を指定しない場合のデフォルト型として `ServerBuilder<Storage, DefaultUser>` が返り、自動的に匿名認証（`AnonymousAuthenticator`）がセットされます。
- 設定値の不整合（FTPS証明書パスの不備など）は、設定メソッド呼び出し時に即 `panic` するのではなく、最後の `build()` メソッドで `Result<Server<...>, ServerError>` として一括検証・返却されます。

## 7. ワークスペース全体での厳格な安全性・品質制約

ルート `Cargo.toml` およびコンパイル検査で、Rustのコンパイラ保証を最大限に活用しています。

- **`unsafe_code = "deny"`**: ワークスペース全体で `unsafe` ブロックの使用を禁止し、メモリ安全性を保証しています。
- **`missing_docs = "deny"` & `clippy::all = "deny"`**: 全ての公開APIへのドキュメント記述と Clippy 警告ゼロを強制しています。
- **`compile_error!` による Feature 検証**: `src/lib.rs` にて、暗号プロバイダ（`aws_lc_rs` または `ring`）のどちらも有効でない場合はコンパイル時に明確なエラーメッセージを出すようになっています。