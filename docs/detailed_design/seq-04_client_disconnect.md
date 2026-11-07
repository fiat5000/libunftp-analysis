# クライアント切断シーケンス図 (`seq-04_client_disconnect.md`)

```mermaid
sequenceDiagram
    autonumber
    actor Client as FTP クライアント
    participant CLoop as ControlChanLoop
    participant DMan as DataChanManager
    participant Session as Session

    activate CLoop
    alt 正常切断: QUIT コマンド受信時
        Client->>CLoop: "QUIT\r\n"
        CLoop->>DMan: 残存データポート・リスナーの解放
        CLoop->>Client: "221 Goodbye.\r\n"
        CLoop->>CLoop: ソケット shutdown(Write) 実行
        Note over CLoop,Client: TCP コネクション切断 (FIN)
    else 異常切断: ネットワーク切断・クライアント強制離脱
        Client-xCLoop: TCP FIN / RST 検知 (read が EOF または Err を返す)
        Note over CLoop: コマンド読み取りエラー検知
    end

    CLoop->>Session: drop(session)
    CLoop->>CLoop: 制御ループ終了
    deactivate CLoop
```

## 図の解説

本図は、FTPクライアントとのセッションが終了する際の2つのパターン（`QUIT` コマンドによる正常切断と、通信断などによる異常切断）およびリソース解放処理の流れを示しています。

### 処理の流れ
1. **正常切断パターン (`QUIT` コマンド受信時 / ステップ 1〜4)**
   - クライアントから `QUIT\r\n` コマンドを受信すると、`ControlChanLoop` はまず `DataChanManager` に指示して、確保済みのパッシブポートや残存するデータチャネルリスナーを解放します。
   - クライアントへ終了応答 `221 Goodbye.\r\n` を送信し、制御ソケットの書き込み側をシャットダウン（TCP FIN送信）してコネクションを正常に閉じます。
2. **異常切断パターン (ネットワーク切断・クライアント強制終了時 / ステップ 5)**
   - クライアントの強制終了やネットワーク障害により TCP FIN（EOF）または RST（リセット）が発生した場合、`ControlChanLoop` のストリーム読み取り処理が `EOF` または I/O エラーを検知してループを抜けます。
3. **セッション破棄とタスク終了 (ステップ 6〜7)**
   - 正常・異常いずれの経路でも、ループ終了時に `Session` および関連チャネル・ソケットが Rust の所有権システム（RAII / `Drop`）によって確実に解放され、クライアント担当の非同期タスクが終了します。
