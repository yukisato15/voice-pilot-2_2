# システムアーキテクチャ

## 全体構成
Zoom Duo Recorder は Electron を用いたデスクトップアプリで、フロント(UI)とバックエンド処理を明確に分離し、Python サブプロセスを通じてメディア処理・アップロード・PDF 生成を行います。

```
+----------------------+        IPC (Electron)       +----------------------+
|   Electron Main      | <-------------------------> |   Renderer (React)   |
|  - アプリライフサイクル |                            |  - UI ロジック          |
|  - Python プロセス制御 |                            |  - 状態管理            |
|  - 設定・永続化        |                            |  - 音声レベル取得       |
+----------+-----------+                            +-----------+----------+
           |                                                     |
           | child_process (JSON over stdout/stderr)             |
           v                                                     |
+----------------------+         gRPC/HTTP(任意)                 |
|   Python Worker      | ----------------------------------------+
|  - ffmpeg ラッパ       |
|  - アップロード I/F     |
|  - PDF 生成           |
|  - 命名・レポート生成    |
+----------------------+
```

## プロセス間通信
- **Main ⇄ Renderer**: Electron IPC (ContextBridge + `ipcRenderer.invoke/on`). UI からの操作コマンド、ステータス更新、モーダル制御などを扱う。
- **Main ⇄ Python**: Node.js `child_process` を利用し、JSON メッセージでリクエスト/レスポンス。長時間処理（ffmpeg, アップロード, PDF 生成）やバックグラウンドキュー操作を委譲する。
- **Python ⇄ ffmpeg**: `subprocess.run`/`subprocess.Popen` によるコマンド実行。ログは stdout/stderr を Main へ中継。

## モジュール構成
- `main`
  - `app.ts`: 起動処理、ウィンドウ生成、起動時モーダル挿入。
  - `ipc-handlers/`: Renderer からの要求受付 (`consent`, `recording`, `themes`, `uploads` など)。
  - `stores/configStore.ts`: `electron-store` による設定保存。
  - `python/processManager.ts`: Python ワーカーの起動・監視。
  - `watchers/zoomWatcher.ts`: `chokidar` ベースの録画フォルダ監視と状態機械。
- `renderer`
  - `screens/OnboardingModal/`: 3 ページ構成の同意モーダル。
  - `screens/PreparationChecklist/`: Zoom 設定/ヘッドホン診断。
  - `screens/SessionControl/`: テーマ提示、タイマー、ヒント、録画インジケータ。
  - `screens/UploadSummary/`: 命名結果とアップロードレポート表示。
  - `components/AudioLevelMeter.tsx`: WebAudio API を利用した入力レベル解析。
  - `hooks/useSessionTimer.ts`: 6 分タイマー、無音カウントダウン、休憩アラート。
- `python`
  - `worker.py`: Node からの JSON コマンドを受信し、各サービスにディスパッチ。
  - `services/ffmpeg.py`: LR 合成、ラウドネス正規化。
  - `services/uploaders/`: S3, SFTP, Box それぞれの実装。
  - `services/pdf.py`: Jinja2 テンプレートのレンダリングと PDF 生成。
  - `services/metadata.py`: 命名規則、メタ JSON、レポート CSV の生成。
  - `services/themes.py`: CSV ロードと重複検出。

## 設定・永続化
- `electron-store` により `config.json` スキーマを保持。初回起動時にデフォルト値を投入。
- 収録セッション管理はインメモリ状態管理（Redux/Recoil など）＋ `better-sqlite3` を利用した軽量キューに拡張可能。
- 同意チェックや操作ログは `app.log` としてローカル保存し、アップロード成否は `upload_report.csv` に記録。

## クロスプラットフォーム考慮
- macOS/Windows 双方で Zoom デフォルトフォルダを自動推定。ユーザーが手動で上書き可能。
- ヘッドホン診断は WebAudio API の `getUserMedia` でマイク入力を取得し、OS 依存実装との差異は UI でガイド。
- ffmpeg はバンドルまたは初回起動時に同意を取りつつ導入。実行パスは設定画面から変更可能にする。
