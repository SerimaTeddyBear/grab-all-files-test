# Grab All Files テストサイト

Google Chrome™ / Microsoft Edge™ 拡張機能「ファイル一括ダウンロード（Grab All Files）」の **全検出パターン** を動作確認するためのテストフィクスチャ集です。

## カバレッジ

- **対応拡張子**: `lib/file-utils.js` の `knownExtRe` に列挙されたすべての対応拡張子（PDF、Office、画像、CAD/3D、科学計算、証明書、フォント、コード設定など）
- **40+ 検出手法**: `popup.js` / `page-interceptor.js` の検出ロジックを網羅

| カテゴリ | カバー内容 | ページ |
|---|---|---|
| HTML 要素 | `a`, `img`, `picture`, `source`, `track`, `iframe`, `embed`, `object`, `canvas` | 01, 02, 03 |
| 属性 | `download`, `type`, `srcset`, `currentSrc`, **25 種の data-\***  | 01, 04 |
| メタタグ | `og:*` (8種), `twitter:*` (6種), `citation_pdf_url` | 05 |
| インラインスクリプト | **8 種の PDF キー** + **11 種の画像キー** + JSON-LD + PDF.js viewer URL | 05, 12 |
| フォーム | POST + `<button>ダウンロード/Download/下载/下載</button>`, hidden input、除外（password/multipart） | 06 |
| 動的読み込み | `blob:` URL, `fetch`, XHR, `URL.createObjectURL` 監視 | 07 |
| URL 推定 | クラウドストレージ (S3/R2/B2/Azure/GCS/Wasabi/DO), LMS (Moodle/Canvas/Blackboard/Liferay), 文書ホスティング (Issuu/Scribd/RG/Academia/SlideShare/DocuSign), `?response-content-type=application/pdf` 等 | 08, 12 |
| プラットフォーム特化 | Gmail (`download_url` / `disp=safe→attd`), Yahoo!メール (`li[data-cy]` / thumbnail), Outlook (`role="option"` / blob:), Salesforce (`069` ContentDocument) | 11 |
| マジックバイト | %PDF, PK, .PNG, JPEG/GIF/WebP/WAV/AVI/MP4/Ogg/FLAC/MP3/RAR/7z/gz/BMP/BZ2/XZ/EXE/ASF | 14 |
| CSS | inline style, computed `backgroundImage` / `borderImageSource` / `listStyleImage`, `image-set()`, `@font-face`, `cursor`, `::before content` | 02, 15 |
| 誤検出フィルタ | SourceForge `/files`/`/reviews`/`/admin`, naturalearthdata `/download/<slug>/`, Liferay AJAX エンドポイント | 09 |
| クロール深度 | depth 0/1/2 で到達できるファイル数の差を確認 | deep/ |

## 構成

```
test-site/
├── index.html                       # ランディング（テストページ一覧）
├── assets/style.css
├── pages/
│   ├── 01-basic.html                # 基本リンク
│   ├── 02-images.html               # 画像（lazy / srcset / CSS背景）
│   ├── 03-embeds.html               # iframe / embed / object
│   ├── 04-data-attrs.html           # data-* 属性 25 種
│   ├── 05-meta-and-script.html      # メタタグ + インラインJS + JSON-LD
│   ├── 06-forms-and-hidden.html     # POST フォーム + hidden input
│   ├── 07-blob-fetch.html           # blob: / fetch / 遅延注入
│   ├── 08-no-extension.html         # クラウド・LMS・CMS の URL 推定
│   ├── 09-misdetection-traps.html   # 除外されるべきナビゲーション URL
│   ├── 10-all-extensions.html       # 全 225 拡張子の網羅リンク
│   ├── 11-platform-mocks.html       # Gmail / Yahoo / Outlook / Salesforce
│   ├── 12-pdfjs-viewer.html         # PDF.js viewer / PDF 閲覧 URL
│   ├── 14-magic-bytes.html          # 拡張子ミスマッチ → header 判定
│   ├── 15-css-scanning.html         # CSS から URL 検出
│   └── deep/{level-1,2,3}.html      # クロール深度テスト
└── files/                           # サンプル実体ファイル (約 230 件)
```

## 公開 URL

| 用途 | URL |
|---|---|
| 本番（QA 用、検索インデックス除外済） | `https://test.grab-all-files.app/` |
| ローカル開発 | `http://localhost/filescanner/test-site/` |

> ⚠ 販売サイト `https://grab-all-files.app/` とは**別ドメイン**です。`test.` サブドメインは検索エンジン除外（`robots.txt` + `<meta name="robots" content="noindex,nofollow">`）の上、Safe Browsing 評価が販売ドメインに波及しないように分離しています。

## デプロイ構成

このリポジトリ（`batch-file-downloader`、private）では GitHub Pages が有効化できないため、`test-site/` の中身を公開 sibling repo `SerimaTeddyBear/grab-all-files-test` にミラーし、そちらの Pages から配信します（販売サイト `SerimaTeddyBear/grab-all-files` と同じパターン）。

```
batch-file-downloader (private)            grab-all-files-test (public)
└─ test-site/             ──sync──▶        └─ (root)            ──Pages──▶  https://test.grab-all-files.app/
   ├─ index.html                              ├─ index.html
   ├─ pages/, files/, assets/                 ├─ pages/, files/, assets/
   ├─ CNAME, robots.txt                       ├─ CNAME, robots.txt
   └─ ...                                     └─ ...
```

ワークフロー: [`.github/workflows/test-site-deploy.yml`](../.github/workflows/test-site-deploy.yml)
（`test-site/**` 変更時のみ実行、または Actions UI から手動）。認証は GitHub Actions シークレット `TEST_SITE_DEPLOY_KEY`（ed25519 Deploy Key の秘密鍵）経由。

セットアップ詳細は [`CWS_SUBMISSION_NOTES.md` の "QA 用サブドメイン" セクション](../CWS_SUBMISSION_NOTES.md#qa-用サブドメイン-testgrab-all-filesapp2026-05-05)を参照。

## ローカルで動かす

XAMPP がインストール済みなので、すでに以下の URL で動作します:

```
http://localhost/filescanner/test-site/
```

Python の組み込みサーバを使う場合:

```powershell
cd c:\xampp\htdocs\filescanner\test-site
python -m http.server 8080
# → http://localhost:8080/
```

> ⚠ `file://` プロトコルでは fetch / blob テスト（07）が CORS で失敗するため、必ず HTTP サーバ経由で開いてください。

## 推奨テストフロー

### 全機能スモークテスト（5 分）

1. `index.html` を開く
2. ポップアップで **深さ = 1** を指定して **Scan** を実行
3. 数百件のファイルが検出されることを確認

### 検出ロジック別の検証

| 確認したいこと | 開くページ | 期待 |
|---|---|---|
| 全拡張子の認識 | `pages/10-all-extensions.html` | 225 件すべて検出 |
| data-* 属性 | `pages/04-data-attrs.html` | 25 件以上検出 |
| メタタグ + インライン JS | `pages/05-meta-and-script.html` | OGP/Twitter/citation/pdf*/image* キー検出 |
| フォーム | `pages/06-forms-and-hidden.html` | POST フォーム 5 件、除外 2 件 |
| blob:/fetch/XHR | `pages/07-blob-fetch.html` | ボタン操作後にファイルが追加 |
| クラウド・LMS | `pages/08-no-extension.html`, `12-pdfjs-viewer.html` | 拡張子なし URL も候補化 |
| プラットフォーム | `pages/11-platform-mocks.html` | download_url / 069 ID 等の構造的検出 |
| マジックバイト | `pages/14-magic-bytes.html` | mystery.bin / archive.dat の種類列が pdf / zip |
| CSS スキャン | `pages/15-css-scanning.html` | background / image-set / @font-face URL 抽出 |
| 誤検出フィルタ | `pages/09-misdetection-traps.html` | SourceForge 等が結果に**現れない** |
| クロール深度 | `pages/deep/level-1.html` | depth 0→1→2 でファイル件数が 1→4→8 |

### ダウンロード機能の確認

1. `pages/01-basic.html` または `pages/10-all-extensions.html` を Scan
2. 保存先フォルダをポップアップにドロップ
3. 全選択 → **ダウンロード**
4. PDF を 3 件以上選択 → **PDF統合**
5. ファイル選択 → **ZIP** でアーカイブ化

## サンプルファイルの中身について

検出 / ダウンロード機能の動作確認用なので、中身は最小サイズです:

- **正規バイナリ**: PDF / PNG / JPG / GIF / WebP / WAV / OLE2 (doc/xls/ppt) / OOXML (docx/xlsx/pptx) / ZIP / RAR / 7z / gz / bz2 / xz / EXE / BMP / TIFF / ICO は適切なマジックバイトを持つ
- **プレースホルダのみ**: HEIC / AVIF / JXL / Camera RAW (CR2/NEF/ARW 等) / その他独自フォーマット → 拡張子の検出はテストできるが、ダウンロード後のファイルとしての妥当性は保証しません

実コンテンツが必要な場合は `files/` 配下の任意のファイルを差し替えてください。

## 注意事項

- `09-misdetection-traps.html` の URL は**結果に現れないこと**が成功条件です
- 拡張機能のソースコード位置:
  - 検出ロジック: `lib/file-utils.js`, `popup.js:2664-3179`
  - ネットワーク監視: `page-interceptor.js`
  - 機能仕様: `docs/sales-feature-document-ja.md`
