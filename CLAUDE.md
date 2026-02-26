# Column Writer — AIコラム執筆アシスタント

## 概要
MAVIS PARTNERS社内向けのAIコラム執筆Webアプリ。過去コラムの文体をAIが学習し、メモから1,500〜3,000字のコラムを自動生成する。単一HTMLファイル（index.html）で動作する。

## 技術構成
- **フロントエンド**: 単一HTML（Vanilla JS、CSS内蔵、フレームワークなし）
- **バックエンド**: Supabase（PostgreSQL + REST API）
- **AI**: Anthropic Claude API（ブラウザから直接呼び出し）
- **デプロイ**: GitHub → Vercel（静的ホスティング）
- **外部ライブラリ**（すべてCDN）:
  - `@supabase/supabase-js` — DB接続
  - `mammoth` — .docxテキスト抽出
  - `pdfjs-dist` — PDFテキスト抽出
  - `docx` — .docxファイル生成（ダウンロード用）

## ファイル構成
```
column-writer/
├── index.html      ← アプリ本体（HTML + CSS + JS すべて含む）
├── CLAUDE.md       ← このファイル
└── setup.sql       ← Supabaseテーブル定義（初回セットアップ用）
```

## 設定の仕組み
### index.html 冒頭の CONFIG（管理者がデプロイ前に記入）
```javascript
const CONFIG = {
  SUPABASE_URL:      'https://xxx.supabase.co',
  SUPABASE_ANON_KEY: 'eyJ...',
};
```
- Supabase接続情報のみHTMLに埋め込み
- **Anthropic APIキーはHTMLに書かない** → ブラウザの管理者画面からDBに保存

### APIキー管理
- `app_settings` テーブルにキーバリューで保存
- ログイン画面下部の「⚙ 管理者設定」→ パスワード認証 → APIキー入力・保存
- 管理者パスワードも同テーブルで管理（初期値: `admin`）

## データベース（Supabase）

### テーブル構造
```
users
├── id          UUID (PK)
├── name        TEXT UNIQUE
├── title       TEXT (肩書き: アナリスト等)
├── password    TEXT
└── created_at  TIMESTAMPTZ

past_columns（文体学習用の過去コラム）
├── id          UUID (PK)
├── user_id     UUID → users.id
├── title       TEXT
├── content     TEXT
├── char_count  INTEGER
└── uploaded_at TIMESTAMPTZ

columns（AI生成したコラム）
├── id              UUID (PK)
├── user_id         UUID → users.id
├── title           TEXT
├── content         TEXT（## 見出し 形式のMarkdown風テキスト）
├── char_count      INTEGER
├── status          TEXT ('draft' | 'complete')
├── original_prompt TEXT
├── created_at      TIMESTAMPTZ
└── updated_at      TIMESTAMPTZ

custom_rules（ユーザーごとの執筆ルール）
├── id          UUID (PK)
├── user_id     UUID → users.id
├── text        TEXT
└── created_at  TIMESTAMPTZ

app_settings（管理者設定）
├── key         TEXT (PK) — 'anthropic_api_key', 'admin_password'
├── value       TEXT
└── updated_at  TIMESTAMPTZ
```

### RLSポリシー
- 全テーブルで `allow_all` ポリシー（社内用簡易認証）
- セキュリティはネットワークレベル（VPN等）で担保する想定

## アプリの画面構成

### ログイン画面
- ユーザー選択 → パスワード入力 → ログイン
- 新規アカウント作成（名前・肩書き・パスワード）
- 管理者設定リンク（APIキー・管理者パスワード変更）
- localStorage (`cw-uid`) で自動ログイン（同一ブラウザ）

### メイン画面（サイドバー + コンテンツ）
| ビュー | state.view | 関数 |
|--------|-----------|------|
| コラム執筆 | `write` | `renderWrite()` |
| 作成済みコラム | `archive` | `renderArchive()` / `renderDetail()` |
| 過去コラム管理 | `past` | `renderPast()` |
| 執筆ルール | `rules` | `renderRules()` |
| 設定 | `profile` | `renderProfile()` |

### コラム詳細画面のアクション
- **⬇ .docx** — `downloadDocx()` でWord文書を生成・ダウンロード
- **🌐 HP用コピー** — `copyWpHtml()` でWordPress用HTMLをクリップボードにコピー
- **📋 テキストコピー** — `copyCol()` でプレーンテキストコピー
- **✏️ 編集** — インラインで直接編集

## AI生成の仕組み

### 関数: `generateColumnAI(prompt, pastCols, rules, draft, feedback)`
- Anthropic Messages API を直接呼び出し（`anthropic-dangerous-direct-browser-access` ヘッダー使用）
- モデル: `claude-sonnet-4-20250514`
- システムプロンプトで過去コラムの文体模倣を指示
- 出力フォーマット:
  ```
  TITLE: タイトル
  ---
  ## セクション見出し1
  本文段落...

  ## セクション見出し2
  本文段落...
  ```

### WordPress HTML変換: `contentToWpHtml(title, content, signature)`
- `## 見出し` → `<h2>見出し</h2>`
- 段落 → `<p>段落</p>`
- 末尾に署名（`MAVIS PARTNERS {肩書き} {名前}`）

### HP風プレビュー: `contentToPreviewHtml(title, content, signature)`
- 目次（セクション見出しリスト）+ 見出し + 段落 + 署名をインラインスタイル付きHTMLで表示

## 主要なグローバル変数
```javascript
db          // Supabase client
apiKey      // Anthropic API key（DBから読み込み）
state       // { currentUser, view, viewingColumn, editMode, users, pastColumns, columns, customRules }
writeState  // { prompt, result, editMode, editTitle, editContent, feedback, generating, revising, error }
loginStep   // 'select' | 'password' | 'create'
loginTarget // ログイン対象ユーザーオブジェクト
```

## デプロイURL
- 本番: https://column-writer.vercel.app
- Supabase: https://jwvsblvfepdfgfuaiayz.supabase.co

## 開発時の注意点
- 単一HTMLファイルのため、関数・CSSがすべて1ファイルに集約されている
- `render()` → `renderApp()` → `renderView()` の描画チェーンで画面を更新
- `switchView()` は `renderApp()` を呼ぶ（サイドバーのアクティブ状態も更新するため）
- ファイルアップロードは `addEventListener` でイベント設定（onclickインラインは使わない）
- ドラッグ＆ドロップ対応済み
