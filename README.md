# Zoom Duo Recorder (Spec v1.0)

Zoom Duo Recorder は、Zoom のローカル録画を前提としたデスクトップ支援アプリです。ヘッドホン装着の確認から録画ガイド、テーマ提示、録音ファイル処理、アップロードまでを一気通貫で実施できるワークフローを設計しています。

## プロジェクトゴール
- Zoom 参加者別 m4a 録音を前提にした 6 分×インターバル 1 分の対話収録支援
- 起動時説明・同意取得（PDF 生成含む）とヘッドホン強制チェック
- 録画開始/停止ガイドと Zoom 録画フォルダ監視
- テーマ提示・ヒント表示・無音指示・タイマー制御
- 収録完了後の命名・メタ生成・オプションで LR 合成・自動アップロード

## 技術スタック概要
- **UI / デスクトップ**: Electron (Node.js 20+) + TypeScript/React を想定
- **バックエンド処理**: Python 3.10+（ffmpeg, アップロード, PDF 生成, ワークフロー制御補助）
- **メディア処理**: ffmpeg 6+
- **主なライブラリ**
  - Node 側: `chokidar`, `electron-store`, `better-sqlite3`(任意), `fast-levenshtein`(任意)
  - Python 側: `boto3`, `paramiko`, `jinja2`, `pdfkit` or `weasyprint`, `pydub`(任意)

## ディレクトリ構成（設計ドキュメント）
- `docs/architecture.md`: システム全体構成と IPC 設計
- `docs/ui-flow.md`: 画面遷移と主要 UI コンポーネント設計
- `docs/data-specs.md`: データディレクトリ命名規則・JSON/CSV スキーマ
- `docs/processes.md`: 録画検出、オーディオ処理、アップロード、同意 PDF 生成の詳細フロー

## 開発環境要件
1. Node.js 20 以上と npm (または pnpm) をインストール
2. Python 3.10 以上と仮想環境ツール (`venv` or `poetry`)
3. ffmpeg 6 以上（macOS/Windows でのバンドルまたは初回セットアップ導入）

## 初期セットアップ（想定）
```bash
# Node 側
cd electron
npm install

# Python 側
cd backend
python -m venv .venv
source .venv/bin/activate  # Windows は .venv\Scripts\activate
pip install -r requirements.txt
```

## サンプルリソース
- `config.example.json`: 初期設定テンプレート。
- `samples/themes_sample.csv`: テーマ CSV サンプル (重複検出テスト用)。
- `templates/consent_template.html`: 同意 PDF 用 Jinja2 テンプレート。

## 実装タスクの概要
- Electron 雛形と設定保存
- 起動モーダルと同意 PDF 生成フロー
- Zoom 録画フォルダ検出と監視
- テーマロード・重複チェック・テーマ提示 UI
- タイマー、ヒント、無音指示、休憩アラート
- メタ/レポート生成、ffmpeg ラッパ、アップロード隊列
- エラーハンドリング・再試行キュー・簡易 E2E テスト

詳細は `docs/` 以下のドキュメントを参照してください。
