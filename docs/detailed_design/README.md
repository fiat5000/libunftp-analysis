# libunftp 内部設計図 説明書 (README.md)

本書は、Rust製FTPサーバーライブラリ「libunftp」のアーキテクチャおよび各ライフサイクル・コマンド処理シーケンスを記述した各Mermaid図（`.md` ファイル）の一覧と概要をまとめたドキュメントです。各図の詳細な解説はそれぞれのファイル内に記載されています。

---

## 📁 ドキュメント一覧

| ドキュメント | 種別 | 概要 |
| :--- | :--- | :--- |
| [`class_diagram.md`](./class_diagram.md) | クラス図 | `libunftp` および関連コアクレート（`unftp-core`）における主要な構造体（Struct）、列挙型（Enum）、トレイト（Trait）の静的関係性を表した内部クラス・構造体設計図 |
| [`seq-01_startup.md`](./seq-01_startup.md) | シーケンス図 | アプリケーションコードが `libunftp` を初期化し、TCPリスナーを開始してリクエスト待機状態に入るまでのサーバー起動シーケンス |
| [`seq-02_shutdown.md`](./seq-02_shutdown.md) | シーケンス図 | OSシグナル（SIGINT/SIGTERM 等）や外部のキャンセルトークンを受け取った際のサーバー停止処理（グレースフルシャットダウン） |
| [`seq-03_client_connect.md`](./seq-03_client_connect.md) | シーケンス図 | クライアントから新規TCP接続が届いてから、専用セッションの初期化・制御タスク分離・初期バナー応答（`220`）が完了するまでのクライアント接続シーケンス |
| [`seq-04_client_disconnect.md`](./seq-04_client_disconnect.md) | シーケンス図 | `QUIT` コマンド受信による正常切断およびネットワーク断（EOF/RST）による異常切断時のクライアント接続終了・リソース解放シーケンス |
| [`seq-05_cmd_auth.md`](./seq-05_cmd_auth.md) | シーケンス図 | `USER` / `PASS` コマンドと `Authenticator` トレイトによるFTP基本認証およびセッション状態更新のシーケンス |
| [`seq-06_cmd_pasv.md`](./seq-06_cmd_pasv.md) | シーケンス図 | `PASV` コマンド受信時に `DataChanManager` がパッシブモード転送待機ポートを割り当ててクライアントへ通知（`227`）するデータチャネル準備シーケンス |
| [`seq-07_cmd_retr.md`](./seq-07_cmd_retr.md) | シーケンス図 | `RETR` コマンドによるファイルダウンロード要求から、`StorageBackend` の非同期ストリーム読み出し・データチャネル転送完了（`226`）までのファイル転送シーケンス |
| [`task-diagram.md`](./task-diagram.md) | アーキテクチャ図 | Tokio ランタイム上で動作する各非同期タスクの階層構造、クライアントとの通信経路（制御／データコネクション）、およびタスク間（`mpsc`/`oneshot`）・プラグイン間の連携を表した構成図 |
