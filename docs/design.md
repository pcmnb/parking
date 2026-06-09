# 車両入出庫管理アプリ 設計書

**バージョン**: 1.0  
**作成日**: 2026-06-10  
**対象リポジトリ**: https://github.com/pcmnb/parking

---

## 1. システム概要

来訪者の車両を管理するWebアプリケーション。受付担当者がスマートフォン・タブレット・PCで操作し、訪問者の入庫・出庫を記録する。

### 主要な設計方針

- **シングルページアプリ**: `index.html` 1ファイルで完結（ビルド不要、依存ライブラリなし）
- **オフライン動作**: データはブラウザのlocalStorageに保存。ネット不通でも入出庫登録・閲覧が可能
- **ゼロバックエンド原則**: OCR機能のみ外部APIを使用し、その他の処理はクライアントで完結
- **ユニバーサルデザイン**: 高齢者・色弱者を考慮した配色・文字サイズ・操作設計

---

## 2. システムアーキテクチャ

```
┌──────────────────────────────────────────────────────────┐
│  ユーザーデバイス（ブラウザ）                              │
│                                                          │
│  index.html（フロントエンド）                             │
│  ├─ 入庫登録フォーム                                      │
│  ├─ 一覧・出庫管理                                        │
│  ├─ CSV入出力                                             │
│  └─ localStorage（データ永続化）                          │
│         ↕ HTTPS POST（画像 base64）                       │
│  Firebase Cloud Functions（OCRプロキシ）                  │
│         ↕ HTTPS POST（OpenAI Responses API）              │
│  OpenAI API（GPT-5.5 マルチモーダル）                     │
└──────────────────────────────────────────────────────────┘
```

### ホスティング

| コンポーネント | 基盤 | URL |
|---|---|---|
| フロントエンド | GitHub Pages | https://pcmnb.github.io/parking/ |
| OCRプロキシ関数 | Firebase Cloud Functions（us-central1） | https://us-central1-parking-2bd69.cloudfunctions.net/ocrPlate |

---

## 3. コンポーネント設計

### 3.1 フロントエンド（index.html）

単一ファイル構成。外部CSS・JSライブラリは一切使用しない。

| 責務 | 実装箇所 |
|---|---|
| ルーティング（タブ切替） | `showView(v)` 関数 |
| 入庫登録 | `register()` 関数 |
| 出庫処理 | `checkout(id, e)` 関数 |
| 一覧表示・検索 | `renderList()` 関数 |
| 編集モーダル | `openEdit(id)` / `saveEdit()` 関数 |
| OCR呼び出し | `handlePhoto()` → `ocrPlate()` 関数 |
| CSV出力 | `doDownloadCSV(mode)` 関数 |
| CSV読み込み | `doImportCSV()` 関数 |
| データ永続化 | `loadRecords()` / `saveRecords()` 関数 |

**タッチ操作の考慮**  
スマートフォンでのスクロール中に誤ってカードをタップしないよう、`touchstart` / `touchend` のY座標差分（10px超で無効化）を判定してから `openEdit()` を呼び出す。

### 3.2 Firebase Cloud Functions（functions/index.js）

| 項目 | 内容 |
|---|---|
| 関数名 | `ocrPlate` |
| トリガー | HTTP（POST） |
| ランタイム | Node.js |
| リージョン | us-central1 |
| 最大インスタンス数 | 10 |
| 認証 | Public（invoker: "public"） |
| CORS | 有効（全オリジン許可） |

**処理フロー**

```
1. POSTリクエスト受信（b64, mediaType）
2. Secret Manager から OPENAI_API_KEY 取得
3. OpenAI Responses API に画像を送信
4. レスポンスから1〜4桁の数字を抽出
5. { number: "1234" } または { number: null } を返却
```

### 3.3 APIキー管理

| 項目 | 内容 |
|---|---|
| Secret名 | `OPENAI_API_KEY` |
| 管理基盤 | Firebase Secret Manager |
| 実際の値 | OpenAI APIキー（GPT-5.5用） |

> **注意**: Secret名が `OPENAI_API_KEY` だが、中身は OpenAI のAPIキー。将来的に `OPENAI_API_KEY` にリネームを推奨（現状は正しい）。

---

## 4. データ設計

### 4.1 localStorageスキーマ

**キー**: `parking_records`  
**値**: JSON配列

```json
[
  {
    "id": 1749480000000,
    "date": "2026-06-10",
    "inTime": "09:30",
    "outTime": "11:45",
    "name": "山田太郎",
    "dest": "営業部 鈴木",
    "plate": "1234"
  }
]
```

| フィールド | 型 | 説明 |
|---|---|---|
| `id` | number | `Date.now()` で生成するユニークID |
| `date` | string | `YYYY-MM-DD` 形式（ローカル日付） |
| `inTime` | string | `HH:MM` 形式（24時間） |
| `outTime` | string \| null | `HH:MM` または `null`（未出庫） |
| `name` | string | 訪問者名・会社名（省略時は「不明」） |
| `dest` | string | 訪問先・担当部署（省略時は「不明」） |
| `plate` | string | ナンバープレートの数字部分（1〜4桁） |

### 4.2 CSVフォーマット

文字コード: UTF-8 BOM付き（Excel互換）  
1行目: ヘッダー行

```
日付,入庫時刻,出庫時刻,訪問者名,訪問先,ナンバー
2026-06-10,09:30,11:45,"山田太郎","営業部 鈴木",1234
2026-06-10,10:00,,"〇〇株式会社","総務部",5678
```

| 列 | フィールド | 備考 |
|---|---|---|
| 日付 | date | `YYYY-MM-DD` |
| 入庫時刻 | inTime | `HH:MM` |
| 出庫時刻 | outTime | 空文字 = 未出庫 |
| 訪問者名 | name | ダブルクォートで囲む |
| 訪問先 | dest | ダブルクォートで囲む |
| ナンバー | plate | 数字のみ |

---

## 5. OCR設計

### 5.1 OCR処理フロー

```
スマートフォンカメラ / ギャラリー
    ↓ File（画像）
FileReader.readAsDataURL()
    ↓ base64文字列
Firebase Cloud Functions（ocrPlate）
    ↓ OpenAI Responses API（vision）
GPT-5.5（画像解析）
    ↓ テキスト（1〜4桁の数字 or 不明）
フロントエンド → plate-number フィールドに自動入力
```

### 5.2 OCRプロンプト設計

- 日本の自動車ナンバープレートから「1〜4桁の数字のみ」を返すよう指示
- 地域名・分類番号・ひらがな・中点は無視
- 読み取れない場合は「不明」を返す
- 数字以外のテキストは返さない（JSONや説明文を排除）

### 5.3 レスポンス処理

| ケース | 処理 |
|---|---|
| `^\d{1,4}$` に完全一致 | そのまま使用 |
| 数字が混在（例: "番号は1234です"） | 正規表現で数字部分を抽出 |
| 「不明」または空文字 | `{ number: null }` を返却 |
| APIエラー | `null` を返却（手動入力を促す） |

---

## 6. セキュリティ設計

| 脅威 | 対策 |
|---|---|
| APIキー漏洩 | フロントエンドにAPIキーを置かず、Cloud Functionsプロキシ経由のみでアクセス |
| 大量リクエスト | `maxInstances: 10` で上限設定。将来的にレートリミット追加を推奨 |
| 不正なbase64データ | Cloud Functions側でAPIキーの有無・画像データの有無を検証 |
| XSS | innerHTML へのユーザー入力は `r.name`, `r.dest`, `r.plate` のみ。plate は数字のみ検証済み。name/dest は登録時にtrimのみ（エスケープなし → 将来対応推奨） |
| データ漏洩 | データはlocalStorageのみ。サーバーへのデータ送信なし（画像のみ送信） |

---

## 7. 非機能要件

| 項目 | 要件 |
|---|---|
| デバイス | スマートフォン・タブレット・PC |
| ブラウザ | モダンブラウザ（Chrome, Safari, Edge） |
| オフライン | OCR機能を除いて全機能が利用可能 |
| レスポンシブ | max-width: 600px でセンタリング |
| 最小文字サイズ | 20px（フィールドラベルは22px） |
| ボタン最小高さ | 48px以上（タップ操作対応） |
| 色覚対応 | 赤緑を識別色に使用しない（青・オレンジ・グレー体系） |
| データ容量 | localStorageの上限（通常5MB）に依存 |

---

## 8. デプロイ構成

```
GitHub リポジトリ（pcmnb/parking）
├── index.html        → GitHub Pages で自動公開
└── functions/
    └── index.js      → firebase deploy --only functions で手動デプロイ
```

**Firebaseプロジェクト**: `parking-2bd69`  
**Firebaseプラン**: Blaze（Cloud Functions利用のため必須）
