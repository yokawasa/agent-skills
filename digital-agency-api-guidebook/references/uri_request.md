# § 2.1 URI設計及びリクエスト

出典: デジタル庁「APIテクニカルガイドブック」§ 2.1

---

## 1. URI設計のポイント [SHOULD]

| ルール | 内容 | 例 |
|--------|------|-----|
| 名詞・複数形を使用 | 動詞・動作を表す言葉は使わない | `city_libraries` ✓ / `getLibrary` ✗ |
| スネイクケース | キャメルケース非推奨 | `city_libraries` ✓ / `cityLibraries` ✗ |
| バージョン番号 | `v` + 整数（小数禁止）、URI に含める | `/v1/`, `/v2/` ✓ / `/v1.2/` ✗ |
| apiサブドメイン | ベースURIに `api` を含める | `https://api.example.go.jp/v1/` |
| ネスト | 階層関係を明確に表現 | `/v1/city_libraries/{user_id}/book/{book_id}` |
| パラメータ指定 | URIの階層が深くなる場合はパラメータで指定 | |
| URI Template形式 | RFC 6570に準拠 | `/v1/magazines/{id}.{format}` |
| 拡張子でデータ形式指定 | 可能 | `/v1/magazines.json`, `/v1/magazines.xml` |
| ベースURIの変更禁止 | 運用開始後は変更しない。変更時は事前に開発者へ通知 | |
| セマンティックバージョニング | バージョニングにはSemantic Versioningを採用 [SHOULD] | |

**URI例:**
```
https://api.example.go.jp/v1/city_libraries
https://api.example.go.jp/v1/city_libraries/{user_id}/book/{book_id}
https://api.example.go.jp/v1/magazines/{id}.{format}
```

---

## 2. HTTPメソッド（CRUD対応） [SHOULD]

| 操作 | メソッド | 備考 |
|------|---------|------|
| Create（生成） | POST | |
| Read（読み取り） | GET | |
| Update（置換） | PUT | |
| Update（部分更新） | PATCH | |
| Delete（削除） | DELETE | |

### コレクション vs 単体

| リソース種別 | メソッド | 操作 |
|------------|---------|------|
| コレクション | GET | 一覧取得 |
| コレクション | POST | 新規作成 |
| 単体 | GET | 個別取得 |
| 単体 | PUT / PATCH | 更新 |
| 単体 | DELETE | 削除 |

---

## 3. パラメータ設計 [SHOULD]

| ルール | 内容 |
|--------|------|
| 複数値の区切り | カンマ `,` を使用 |
| ページネーション | `limit`（件数） + `offset` または `page`（開始位置）、初期値100件以下推奨 |
| フィールド絞り込み | 返却項目が10件以上の場合は `fields` パラメータを実装 |

### 標準パラメータ一覧

| 名称 | 概要 |
|------|------|
| `limit` | 1回のリクエストで返却するデータ件数（`per_page` より `limit` を推奨） |
| `offset` | 先頭から指定件数をスキップして返却 |
| `page` | 返却データの開始位置（`limit` と組み合わせて使用） |
| `since` | 指定日付以降のデータを返却 |
| `until` | 指定日付以前のデータを返却 |
| `sort` | 並び替え条件を指定 |
| `encode` | 文字コードを指定 |
| `fields` | 指定した項目のみを返却 |
| `type` | 返却フォーマットを指定（URIでの指定を推奨） |

---

## 4. 同期API vs 非同期API

### 選択基準

| 条件 | 推奨 |
|------|------|
| 処理時間 < 1秒、即時結果が必要、リソースの即時状態変更が必要 | 同期API |
| 処理時間 ≥ 1秒、バックグラウンド処理可能、即時結果不要 | 非同期API |

### 同期API実装規約 [MUST]
- 成功時: `200 OK`、レスポンスボディに直接結果を含める
- タイムアウト設定とクライアントへの通知（例: `504 Gateway Timeout`）

### 非同期API実装規約 [MUST]

**リクエスト受付時:**
```
HTTP/1.1 202 Accepted
Location: https://api.example.go.jp/v1/status/{job_id}
```

**ステータス確認エンドポイント:**
```
GET https://api.example.go.jp/v1/status/{job_id}
→ 処理中: 202 Accepted
→ 完了:   303 See Other + 結果URL
```

**Webhook:** 可能であれば完了通知の Webhook サポートを検討 [MAY]

---

## 5. データ形式 [SHOULD]

| ルール | 内容 |
|--------|------|
| 文字コード | **UTF-8** [MUST]。元データがShift_JIS等の場合は変換・縮退を行う |
| マルチバイト文字 | JIS X 0213（第1〜第4水準）を利用 [MUST] |
| 外字 | リクエストデータに使用禁止 [MUST] |
| 非英字の命名 | スネイクケースを使用（ただしURIドメイン名の制約に留意） |
| パーセントエンコーディング | URIに含めない [SHOULD] |
| リクエスト形式 | 単純なパラメータ値またはカンマ区切り配列を推奨 [SHOULD] |
