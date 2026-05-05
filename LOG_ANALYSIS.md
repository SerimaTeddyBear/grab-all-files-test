# スキャン結果ログ分析ガイド

## 概要

拡張機能の検出ロジックがテストフィクスチャの仕様通りに動作しているかを検証するため、ポップアップに「**スキャンログ出力 (JSON)**」ボタンが追加されています。スキャン結果一式（メタデータ・統計・全アイテム・エラー）を JSON ファイルにエクスポートして、Claude や他の解析ツールに渡せます。

ソース: [popup.html:212](../popup.html#L212), [popup.js:6751](../popup.js#L6751)

## 使い方

1. テストサイトの任意のページを開く（例: `http://localhost/filescanner/test-site/pages/04-data-attrs.html`）
2. 拡張機能アイコンをクリック → ポップアップ
3. 検索したいファイル種類を選択
4. **Scan** を実行
5. 結果一覧の上部にある **📊 スキャンログ出力 (JSON)** ボタンをクリック
6. `test-site/logs/` 配下に保存（推奨）

ファイル名: `gaf-log_<host>_<timestamp>.json`

## JSON スキーマ (schema_version 2)

```jsonc
{
  "schema_version": 2,
  "exported_at": "2026-05-03T07:42:11.000Z",
  "user_agent": "Mozilla/5.0 ...",

  "extension": {
    "name": "ファイル一括ダウンロード",
    "version": "4.2.1",
    "manifest_version": 3
  },

  // ─── セッション識別子（複数のテストログをグルーピング）───
  "session": {
    "device_id": "uuid-...",        // インストール時に生成
    "install_date_ms": 1714680000000,
    "trial_expired": false,
    "trial_days_remaining": 12,
    "license_active": true,
    "last_verified_ms": 1714780000000,
    "last_trial_check_ms": 1714780000000
  },

  // ─── スキャン実行コンテキスト ───
  "scan": {
    "mode": "page",                                     // "page" or "url"
    "source_url": "http://localhost/.../04-data-attrs.html",
    "source_title": "04. data-* 属性 — ...",
    "source_tab_id": 12345,
    "source_tab_window_id": 100,
    "source_tab_incognito": false,
    "source_tab_status": "complete",

    "started_at": "2026-05-03T07:42:08.123Z",          // スキャン開始
    "ended_at":   "2026-05-03T07:42:09.456Z",          // スキャン終了
    "duration_ms": 1333,                                // 経過 ms

    "depth": "0",                                       // 深さセレクタの値（"0"="このページのみ", "1+"=クロール）

    "file_types_selected_count": 225,                   // 選択中の拡張子数
    "file_types_total": 225,                            // 対応拡張子総数
    "file_types_selected": ["3fr", "3g2", "..."],

    // ─── Deep Scan 時のクロールサマリー（depth >= 1 で populated）───
    "crawl_summary": {
      "target_url": "http://...",
      "max_pages_cap": 1000,
      "pages_scanned": 4,
      "pages_queued_at_end": 0,
      "fetch_errors": 0,
      "cap_reached": false,
      "max_depth": 2,
      "files_found": 8
    },

    // ─── スキャン中に検出された各種エラー ───
    "errors": [
      { "at": 1714780000000, "kind": "bot_wall_cloudflare", "message": "..." },
      { "at": 1714780000000, "kind": "scan_timeout", "message": "..." },
      { "at": 1714780000000, "kind": "partial_fetch_errors", "message": "3 page(s) failed during crawl" },
      { "at": 1714780000000, "kind": "restricted_url", "message": "..." }
      // kind: scan_failed | restricted_url | restricted_page | scan_timeout
      //     | network_error | frame_not_ready | bot_wall_cloudflare
      //     | partial_fetch_errors
    ]
  },

  // ─── 表示時フィルタの状態 ───
  "filters": {
    "name_filter": "",                  // ファイル名/タイトル絞り込み
    "size_min": "",                     // サイズ下限
    "size_max": "",
    "ext_chips_selected": ["pdf", "docx"]    // 拡張子チップで選択中
  },

  // ─── 統計サマリー ───
  "stats": {
    "total_detected": 25,               // 全検出数
    "shown_after_filters": 25,          // フィルタ後の表示数
    "by_extension": { "pdf": 4, "docx": 1, "...": 1 },
    "by_detection_source": {
      "dom": 18,                        // DOM スキャナで発見
      "network": 0,                     // fetch/XHR インターセプト
      "browser-download": 0,            // ブラウザ側ダウンロード監視
      "current-url": 0,                 // 現在のタブ URL 自身がファイル
      "blob-element": 0,                // blob: URL（Outlook 等）
      "form-post": 0                    // POST フォーム
    },
    "by_type_hint": { "pdf": 5, "jpg": 8 },     // typeHint 別の内訳
    "with_type_hint": 13,
    "broken_links": 0,
    "probe_errors": 0,
    "downloads_completed": 0,
    "downloads_failed": 0
  },

  // ─── 検出された各アイテム ───
  "items": [
    {
      "url": "http://.../files/sample.pdf",
      "title": "data-href",
      "filename": "sample.pdf",
      "ext": "pdf",
      "type_hint": null,                  // "pdf" / "jpg" / "mp4" 等の型ヒント
      "mime_type": null,                  // ネットワーク検出時の MIME
      "frame_id": null,                   // 子フレーム由来なら frameId
      "detection_source": "dom",          // "dom" / "network" / "blob-element" / "form-post" / ...
      "post_form": null,                  // {method, fields, label} or null
      "blob_download": false,             // blob: 由来か
      "referer": "http://...",
      "dom_index": 12,                    // DOM 出現順 (sortable)
      "size_bytes": 494,                  // HEAD 取得済みなら数値
      "modified": null,                   // Last-Modified ヘッダ
      "in_filtered_view": true,           // チップ/絞り込みで表示対象か
      "checked_for_download": true,       // チェックボックスON
      "probe_error": null,                // {status, kind} HEADで取得失敗時 (404/403/410)
      "broken_link": false,               // 上記により壊れリンク扱い
      "download_status": null,            // null / "dl-pending" / "dl-active" / "dl-complete" / "dl-failed"
      "download_error": null              // {key, status, raw} ダウンロード失敗時
    }
  ]
}
```

## 分析の観点

### 1. 検出漏れチェック (page レベル)

ページごとの期待検出件数:

| ページ | 期待件数 | 主な確認項目 |
|---|---|---|
| `pages/01-basic.html` | 18 | 全拡張子カテゴリの基本リンク |
| `pages/02-images.html` | 14+ | lazy / srcset / picture / CSS background |
| `pages/03-embeds.html` | 8 | iframe / embed / object |
| `pages/04-data-attrs.html` | 25+ | 全 25 種の data-* 属性 |
| `pages/05-meta-and-script.html` | 30+ | OGP/Twitter/citation + pdfUrl 17 キー |
| `pages/06-forms-and-hidden.html` | 5 (＋除外2) | POST フォーム検出 |
| `pages/07-blob-fetch.html` | 動的 | スキャン後ボタン押下で増加 |
| `pages/08-no-extension.html` | 30+ | 拡張子なし URL の推定検出 |
| `pages/09-misdetection-traps.html` | **0** | 除外フィルタが機能しているか |
| `pages/10-all-extensions.html` | 225 | 全対応拡張子 |
| `pages/11-platform-mocks.html` | 12+ | Gmail/Yahoo/Outlook/Salesforce |
| `pages/12-pdfjs-viewer.html` | 30+ | PDF.js viewer URL パターン |
| `pages/14-magic-bytes.html` | 6 | 拡張子ミスマッチ判定 |
| `pages/15-css-scanning.html` | 15+ | image-set / border-image / @font-face |

### 2. 誤検出チェック

`pages/09-misdetection-traps.html` の log:
- `stats.total_detected === 0` であるべき
- `nonFileUrlRe` フィルタが機能していれば SourceForge / naturalearthdata / Liferay AJAX URL は含まれない

### 3. type_hint 付与の正確性

| 検出元 | 期待される type_hint |
|---|---|
| `data-pdf-url` | `"pdf"` |
| `data-zoom-image` / `data-thumbnail` / `data-bg` 等 | `"jpg"` |
| `meta og:image` | `"jpg"` |
| `meta citation_pdf_url` | `"pdf"` |

`stats.by_type_hint` の分布を `pages/04-data-attrs.html` / `pages/05-meta-and-script.html` で検証。

### 4. POSTフォーム検出

`pages/06-forms-and-hidden.html` で:
- `stats.by_detection_source["form-post"] === 5`
- 除外されるべきパスワード付きフォーム / multipart は **含まれない**
- 各 form-post アイテムには `post_form: { method, fields, label }` が入る

### 5. クロール深度

`pages/deep/level-1.html` を depth 0 → 1 → 2 で実行し、それぞれの log で:
- `scan.crawl_summary.pages_scanned` が depth に応じて 1 → 3 → 4
- `stats.total_detected` が 1 → 4 → 8
- `scan.crawl_summary.fetch_errors === 0`

### 6. ネットワーク検出 vs DOM 検出

`stats.by_detection_source` の分布で:
- 静的ページ: `dom` がほぼ全数
- `pages/07-blob-fetch.html` で動的ボタンを押した後: `blob-element` / `network` が増える
- Gmail/Outlook 風モック (`pages/11-platform-mocks.html`): `dom` だが `blob_download: true` のアイテムが含まれる

### 7. パフォーマンス

`scan.duration_ms` で:
- 静的ページの depth 0: 数百ms 以内が望ましい
- 異常に長い場合は `scan.errors` に `scan_timeout` がないか確認

### 8. 壊れリンク／プローブエラー

- `pages/08-no-extension.html` のような外部ドメイン参照は `probe_error.status: 404` などになる場合がある
- `stats.broken_links` が異常に多いとフィクスチャ側のリンク切れ可能性

## Claude に渡す手順

1. ログ JSON を `test-site/logs/` に保存
2. このプロジェクトでチャットを開いて:
   ```
   このログを test-site/LOG_ANALYSIS.md の「分析の観点」に沿って検証してください:
   c:/xampp/htdocs/filescanner/test-site/logs/gaf-log_localhost_2026-05-03T07-42-09.json
   ```
3. Claude が期待値との差分・検出漏れ・誤検出・type_hint 不整合・タイミング異常などを表形式で報告します

## 限界事項（現バージョンで取れていないデータ）

以下は将来の拡張で追加する候補:

- **DOM 検出のサブ分類**: `dom` をさらに `anchor` / `img` / `iframe` / `data-pdf-url` / `meta` / `script` 等に細分化するには、`addUrl` 全 15 箇所への source 引数追加が必要。現状は **テストページの URL（フィクスチャ側）で検出元を識別**できるよう設計されています（例: `pages/04-data-attrs.html` のアイテムは全部 data-* 由来とみなしてよい）。
- **棄却 URL の理由付きログ**: nonFileUrlRe で弾かれた URL を別配列で残す機能は未実装（現状は `pages/09-misdetection-traps.html` のスキャン結果が空であることでのみ確認可能）。
- **per-page クロールトレース**: depth 2+ で実際にどのページをどの順で訪問したかは pages_scanned カウントのみ。各ページのレスポンスサイズ・タイミングは取れていません。
- **HEAD probe の詳細エラー**: `probe_error` に HTTP status は入りますが、ネットワーク層のエラー（タイムアウト・DNS 等）は捕捉していません。

これらが必要になったら相談ください — それぞれ追加できます。
