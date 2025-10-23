# Voice Pilot (Zoom収録ディレクター補助ツール)

Zoom による 2 者対話収録を支援する Electron + React アプリです。ディレクターが操作する `DirectorConsole` と、参加者向けの共有画面 `SharedStage` を並行して出力し、タイマー・ヒント・指示テキスト・出来事ログ・Zoom 録画管理をまとめて扱えます。

## 主な機能
- タイマーのカウントアップ/ダウン切り替え、オーバータイム警告、カウントダウン演出
- テーマ/スライド読み込みと SharedStage でのヒント・指示・イベントピル表示
- 出来事ログ・指示メモの記録、CSV/JSONL/FCPXML へのエクスポート
- Zoom ローカル録画フォルダ監視と自動リネーム、録画押し忘れ検知
- 設定項目 (セッション長、無音秒数、FPS、アラート時間 等) の即時反映

## リポジトリ構成
- `app/main`: Electron メインプロセスと IPC ハンドラ、Zoom/エクスポートユーティリティ
- `app/renderer`: React + Vite で構築した DirectorConsole / SharedStage UI
- `backend/`: Python ワーカー (音声処理・アップロード補助)
- `config*.json`: 収録設定テンプレート
- `docs/`: 設計メモとワークフロー仕様
- `samples/`: テーマ CSV やダミー素材
- `electron/`: 旧構成 (参照用)。現行開発は `app/` 配下を利用します。

## セットアップ
1. Node.js 20 以上と npm をインストール
2. 依存パッケージをインストール
   ```bash
   npm install --prefix app
   ```
3. 初期設定が必要な場合は `config.example.json` を参考に `config.json` を調整し、必要なら Python 側 (`backend/`) の仮想環境も用意します。

## 起動と動作確認
1. 開発サーバーを起動
   ```bash
   npm run dev --prefix app
   ```
2. IPC ログを追いたい場合は別ターミナルで
   ```bash
   npm run dev:main --prefix app
   ```
3. アプリ起動後のチェックフロー
   - `DirectorConsole` からテーマ CSV / スライドを読み込み、`開始` ボタンでタイマーをスタート
   - `SharedStage` の右パネル (ヒント)、上部 (指示)、左下 (イベントピル)、下部 (録画ステータス) を確認
   - 保存先フォルダと Zoom 録音フォルダを指定し、Zoom 側で録画開始 → 停止してファイルがリネームされるか検証
   - 必要であれば Zoom の仮録画ファイルを投入してリネーム挙動をテスト

## SharedStage の表示
- 指示テキストはグラデーションバッジで表示され、自動消滅までの残り時間を示します。長文でも 38vh までスクロール表示に対応。
- イベントピルは左下に配置し、カテゴリ色とパルスドットで直感的に把握できます。
- ヒントパネルは最大 65vh までを固定し、縦スクロールと最新利用ハイライト・使用済み色分けを実装しています。
- 録画状態、タイマー、テーマ情報、ロール別プロンプトを一画面で把握できます。
- 残り 10 秒になると赤い全画面オーバーレイでカウントダウンし、0 秒で STOP 表示に切り替わります。

## ガイダンススライドの準備
- `PPT/画像フォルダを読み込む…` ボタンから、PNG/JPEG/WEBP 画像をまとめたフォルダを指定できます。
- PowerPoint からは「ファイル > エクスポート > 画像」で各スライドの画像を書き出し、ファイル名（例: slide_01.png）の昇順で並べてください。
- フォルダ読み込み時はファイル名順でソートし、最大枚数の制限なく SharedStage に順送り表示します。
- 動作確認用にサンプルスライドも同じパネルから読み込めます。

## 操作用ショートカット
- `F1`〜`F6`: 各出来事カテゴリ (個人情報 / 外部ノイズ / 参加者A / 参加者B / 通信トラブル / NGワード)
- `[` / `]`: カット IN / カット OUT のマーク挿入
- `H`: 次のヒントを SharedStage に表示 (再利用で順次進行)
- それ以外のキーは入力フォームがアクティブな場合を除き無視されます。

## 主要設定項目
`app/main/config/store.ts` の Electron Store (`voice-pilot-config.json`) と `DirectorConsole` の設定セクションで編集できます。
- `projectDir`: エクスポート先のプロジェクトフォルダ
- `zoomRecordingDir`: Zoom ローカル録画フォルダ
- `projectCode` / `pairId` / `segmentCounter`: リネーム時のファイル命名に使用
- `timerMode`: `up` / `down` の切り替え
- `sessionLength`, `startSilence`, `endSilence`: タイマー初期値と無音挿入秒
- `intervalSeconds`, `breakEveryMinutes`, `breakLengthMinutes`: インターバルと休憩設定
- `endWarnSec`: SharedStage の終了警告秒数
- `memoDisplaySec`, `eventPillSec`: メモ・イベント表示時間
- `timecodeFps`: エクスポート時のタイムコード FPS
- `overtimeAlert`: オーバータイム時の警告表示 ON/OFF
- `showProjectSelector`: 起動時のプロジェクトフォルダ選択ウィザード表示

## ログ・エクスポート
- 出来事ログ／メモは CSV, JSONL, FCPXML として `DirectorConsole` からエクスポート可能です。
- プロジェクトフォルダが設定されている場合は `PROJECT_DIR/exports/yyyymmdd_HHMMSS/` に自動で保存されます。
- プロジェクトフォルダが未設定 or 参照不可の場合は保存先選択ダイアログを表示し、トーストでフォールバックを通知します。
- Zoom 録画が完了すると自動で命名され、`config` の各種カウンタが更新されます。

## テスト
- エクスポートフォーマットは `tsx` を利用した Node.js テストで検証できます。
  ```bash
  npm run test:exports --prefix app
  ```
  サンドボックス環境で一時ディレクトリ作成に制限がある場合は `TMPDIR=$(pwd)/.tmp` を一時的に指定してください。

## 補足メモ
- IPC 呼び出しを拡張したため、開発中は `npm run dev:main --prefix app` のログを常に確認してください。
- `config/store.ts` に設定キーを追加する際は初期値と Electron Store のスキーマを整合させること。
- SharedStage に表示するショートカット一覧やガイドは今後 UI 側で追記予定です。
