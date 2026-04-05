# § 2.2 レスポンス

出典: デジタル庁「APIテクニカルガイドブック」§ 2.2

---

## 1. HTTPステータスコード一覧

### 成功系

| コード | 名称 | 使用場面 |
|--------|------|---------|
| 200 | OK | GETリクエストの成功、更新操作の成功 |
| 201 | Created | POST/PUTによる新規リソース作成 |
| 202 | Accepted | 長時間処理の受付（非同期APIの開始時） |
| 204 | No Content | DELETEリクエストの成功（返すコンテンツなし） |
| 206 | Partial Content | 大容量データの分割ストリーミング（ストリーミングAPI） |
| 303 | See Other | タスク完了後、結果URLへのリダイレクト（非同期APIの完了時） |

### エラー系（クライアント）

| コード | 名称 | 使用場面 |
|--------|------|---------|
| 400 | Bad Request | 不正なリクエストパラメータ、無効なデータ形式 |
| 401 | Unauthorized | APIキー不在、無効なアクセストークン |
| 403 | Forbidden | 権限のないリソースへのアクセス試行 |
| 404 | Not Found | 存在しないエンドポイントやリソースID |
| 405 | Method Not Allowed | 許可されていないHTTPメソッドの使用 |
| 408 | Request Timeout | クライアントが長時間無応答（ストリーミングAPI） |
| 409 | Conflict | 同時更新による競合 |
| 422 | Unprocessable Entity | バリデーションエラー |
| 429 | Too Many Requests | レート制限の超過 |

### エラー系（サーバー）

| コード | 名称 | 使用場面 |
|--------|------|---------|
| 500 | Internal Server Error | サーバー側の実行時エラー |
| 503 | Service Unavailable | メンテナンス中、過負荷状態 |

---

## 2. 主要レスポンスヘッダー

| ヘッダー名 | 説明 | 例 |
|-----------|------|-----|
| `Access-Control-Allow-Origin` | CORSポリシー | `*` |
| `Content-Type` | レスポンスのメディアタイプ | `application/json; charset=utf-8` |
| `Cache-Control` | キャッシュ動作の指定 | `max-age=3600, public` |
| `ETag` | リソースの特定バージョン識別子 | `"33a64df551..."` |
| `Link` | ページネーションのための関連リンク | `<https://api.example.com/users?page=2>; rel="next"` |
| `Location` | 新しく作成されたリソースのURI | `/users/1234` |
| `Strict-Transport-Security` | HTTPS使用の強制 | `max-age=31536000; includeSubDomains` |
| `X-Request-ID` | リクエストの一意識別子 | `550e8400-e29b-41d4-a716-446655440000` |
| `X-RateLimit-Limit` | 期間あたりの最大リクエスト数 | `5000` |
| `X-RateLimit-Remaining` | 残りリクエスト可能数 | `4999` |
| `X-RateLimit-Reset` | レート制限リセット時刻（Unix時間） | `1372700873` |
| `X-Response-Time` | APIレスポンス時間 | `87ms` |

---

## 3. 正常レスポンス（WebAPI）[SHOULD]

### データ形式の優先度

| 形式 | 推奨度 | 備考 |
|------|--------|------|
| JSON | **推奨** | 軽量・普及度高。RFC 4627 |
| XML | 環境が整備済みの場合に使用可 | W3C準拠 |
| CSV | 利用者利便性が高い場合 | RFC 4180 |
| バイナリ（zip等） | 申請書・公文書等の場合を除き**使用しない** | |

### 専用データ形式（分野別）

| データ種別 | 推奨形式 | 規格 |
|-----------|---------|------|
| 地理空間情報 | GeoJSON | RFC 7946 |
| 地理空間情報（代替） | KML | Open Geospatial Consortium（測地系サポートなし注意） |
| スケジュール・イベント | iCalendar | RFC 5545 |
| 連絡先 | vCard | RFC 6350 |

### レスポンスJSONのメタデータ構造

```json
{
  "metadata": {
    "title": "書籍情報取得",
    "detail": "正常実行されました。",
    "type": "https://example.go.jp/docs/",
    "parameter": {
      "offset": 25,
      "limit": 25
    },
    "resultset": {
      "count": 227,
      "offset": 25,
      "limit": 25
    }
  },
  "results": []
}
```

### 代表的なメタデータ項目

| 名称 | 概要 |
|------|------|
| `status` | HTTPステータスコード（レスポンスヘッダと同値） |
| `type` | APIドキュメントのURL |
| `title` | WebAPIの機能名称 |
| `detail` | 処理結果の説明 |
| `instance` | リクエストURL |
| `parameter` | APIに渡されたパラメータ |
| `resultset` | 全件数・返却件数等のメタ情報 |
| `result` | 結果データ |

### JSON+HAL形式（リソース間リンク）[MAY]

手続APIなど一連の処理フローがある場合、HALによるリンク表現を検討する。

```json
{
  "_links": {
    "self": { "href": "/orders" },
    "next": { "href": "/orders?page=2" },
    "find": { "href": "/orders{?id}", "templated": true }
  },
  "_embedded": {
    "orders": [{
      "_links": {
        "self": { "href": "/orders/123" },
        "basket": { "href": "/baskets/98712" },
        "customer": { "href": "/customers/7809" }
      },
      "total": 30000,
      "currency": "JPY",
      "status": "出荷中"
    }]
  },
  "currentlyProcessing": 14,
  "shippedToday": 20
}
```

---

## 4. エラーレスポンス（WebAPI）[SHOULD]

RFC 7807「Problem Details for HTTP APIs」に準拠。

### 必須項目

| 名称 | 概要 |
|------|------|
| `status` | HTTPステータスコード |
| `type` | エラー種別を示すURI。該当なしの場合は `"about:blank"` |
| `title` | エラーの名称 |
| `detail` | エラーの説明（API利用者が問題箇所を理解できる内容） |
| `instance` | エラーが発生したURI |

### エラーレスポンス例

```http
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json
Content-Language: ja

{
  "type": "https://example.com/probs/out-of-credit",
  "title": "口座残高が十分でありません。",
  "detail": "口座残高は3000円です。5000円の支払いが必要です。",
  "instance": "/account/12345/msgs/abc",
  "balance": 3000,
  "accounts": ["/account/12345", "/account/67890"]
}
```

> 推奨項目（`status`〜`instance`）に加え、アプリケーション固有の拡張項目（`balance`等）を追加可能 [MAY]

---

## 5. 非同期APIのレスポンス

### ① リクエスト受付時 [MUST]
```json
HTTP/1.1 202 Accepted
Location: /api/v1/tasks/123e4567-e89b-12d3-a456-426614174000

{
  "task_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "processing",
  "status_url": "/api/v1/tasks/123e4567-e89b-12d3-a456-426614174000"
}
```

### ② 状態確認エンドポイント
```json
{
  "task_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "completed",
  "result_url": "/api/v1/results/987fgh-e89b-12d3-a456-426614174000"
}
```

### ③ エラー時
```json
{
  "task_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "failed",
  "error": {
    "code": "PROC_ERROR",
    "message": "データ処理中にエラーが発生しました。"
  }
}
```

### ④ タイムアウト・完了通知
- 長時間タスクには適切なタイムアウト設定と挙動定義 [MUST]
- 可能であればWebhook等による完了通知をサポート [MAY]

---

## 6. ストリーミングAPIのレスポンス設計 [SHOULD]

| 観点 | 実装指針 |
|------|---------|
| コネクション確立 | HTTP/1.1: `Transfer-Encoding: chunked`。WebSocket使用時は適切なハンドシェイクを実装 |
| データフォーマット | JSON Lines形式（各行が有効なJSON）を推奨 |
| エラー処理 | エラー発生時は特別なフォーマットでエラーメッセージを送信 |
| 接続管理 | 切断時の再接続メカニズム、サーバー側タイムアウト処理を実装 |
| バックプレッシャー | クライアントがデータを処理しきれない場合のメカニズムを実装 |
| セッション終了 | ストリーム終了を明示的に示す（特別な終了メッセージ等） |
| サンプルレート制御 | 必要に応じてクライアントが送信レートを制御できるメカニズムを提供 [MAY] |

### JSON Lines形式例
```json
{"id": 1, "data": "チャンク1"}
{"id": 2, "data": "チャンク2"}
{"id": 3, "data": "チャンク3"}
```

### ストリーミングエラー例
```json
{"error": true, "code": "STREAM_ERROR", "message": "データ送信中にエラーが発生しました。"}
```
