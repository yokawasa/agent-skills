# 住民票一括データ変換処理 非同期API仕様

> デジタル庁「APIテクニカルガイドブック（DS-464-2, 2024年9月30日版）」に基づく設計

---

## 概要

住民票の一括データ変換処理は数分かかる処理であり、処理時間が1秒以上かつバックグラウンド処理が可能なケースに該当します。デジタル庁ガイドラインに従い、**非同期API**として設計します。

---

## 1. 処理開始エンドポイント

### URI・メソッド

```
POST https://api.example.go.jp/v1/resident_records/batch_conversions
```

- **[SHOULD]** URI はリソースを表す名詞・複数形・snake_case を使用
- **[SHOULD]** バージョンを `/v1/` として URI に含める
- **[SHOULD]** サブドメインに `api.` を使用
- HTTP メソッドは新規ジョブ作成なので `POST` を使用 **[MUST]**

### リクエストヘッダー

```
POST /v1/resident_records/batch_conversions HTTP/1.1
Host: api.example.go.jp
Content-Type: application/json
Authorization: Bearer <access_token>
```

### リクエストボディ

```json
{
  "source_format": "xml",
  "target_format": "json",
  "record_ids": [
    "REC-00001",
    "REC-00002",
    "REC-00003"
  ],
  "options": {
    "encoding": "UTF-8",
    "include_history": false
  }
}
```

### レスポンス（処理受付時）**[MUST]**

```
HTTP/1.1 202 Accepted
Content-Type: application/json
Location: https://api.example.go.jp/v1/resident_records/batch_conversions/status/123e4567-e89b-12d3-a456-426614174000
```

```json
{
  "task_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "processing",
  "status_url": "/v1/resident_records/batch_conversions/status/123e4567-e89b-12d3-a456-426614174000",
  "accepted_at": "2026-04-12T10:00:00+09:00",
  "estimated_completion_seconds": 180
}
```

- **[MUST]** `202 Accepted` を返す（処理はバックグラウンドで継続）
- **[MUST]** `Location` ヘッダーにステータス確認 URL を含める
- **[SHOULD]** `task_id` をレスポンスボディに含め、以降の問い合わせに使用可能にする

---

## 2. ステータス確認エンドポイント

### URI・メソッド

```
GET https://api.example.go.jp/v1/resident_records/batch_conversions/status/{task_id}
```

### リクエストヘッダー

```
GET /v1/resident_records/batch_conversions/status/123e4567-e89b-12d3-a456-426614174000 HTTP/1.1
Host: api.example.go.jp
Authorization: Bearer <access_token>
```

### レスポンス：処理中

```
HTTP/1.1 202 Accepted
Content-Type: application/json
```

```json
{
  "task_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "processing",
  "progress_percent": 45,
  "started_at": "2026-04-12T10:00:01+09:00",
  "estimated_completion_seconds": 100
}
```

- **[MUST]** 処理中は `202 Accepted` を返す

### レスポンス：処理完了

```
HTTP/1.1 303 See Other
Content-Type: application/json
Location: https://api.example.go.jp/v1/resident_records/batch_conversions/results/123e4567-e89b-12d3-a456-426614174000
```

```json
{
  "task_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "completed",
  "result_url": "/v1/resident_records/batch_conversions/results/123e4567-e89b-12d3-a456-426614174000",
  "completed_at": "2026-04-12T10:03:15+09:00"
}
```

- **[MUST]** 処理完了時は `303 See Other` を返す
- **[MUST]** `Location` ヘッダーに結果取得 URL を含める

### レスポンス：処理失敗

```
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "task_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "failed",
  "error": {
    "code": "PROC_ERROR",
    "message": "データ処理中にエラーが発生しました。",
    "detail": "変換対象のレコード REC-00002 のフォーマットが不正です。"
  },
  "failed_at": "2026-04-12T10:01:30+09:00"
}
```

---

## 3. 結果取得エンドポイント

### URI・メソッド

```
GET https://api.example.go.jp/v1/resident_records/batch_conversions/results/{task_id}
```

### リクエストヘッダー

```
GET /v1/resident_records/batch_conversions/results/123e4567-e89b-12d3-a456-426614174000 HTTP/1.1
Host: api.example.go.jp
Authorization: Bearer <access_token>
```

### レスポンス

```
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "task_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "completed",
  "total_records": 3,
  "converted_records": 3,
  "results": [
    {
      "record_id": "REC-00001",
      "status": "success",
      "output_url": "/v1/resident_records/REC-00001/converted"
    },
    {
      "record_id": "REC-00002",
      "status": "success",
      "output_url": "/v1/resident_records/REC-00002/converted"
    },
    {
      "record_id": "REC-00003",
      "status": "success",
      "output_url": "/v1/resident_records/REC-00003/converted"
    }
  ],
  "completed_at": "2026-04-12T10:03:15+09:00"
}
```

---

## 4. 全体フローとHTTPステータスコード一覧

### シーケンス図（テキスト形式）

```
クライアント                              サーバー
    |                                        |
    |-- POST /v1/resident_records/           |
    |         batch_conversions -----------> |
    |                                        | (ジョブをキューに追加)
    |<-- 202 Accepted                        |
    |    Location: .../status/{task_id}      |
    |                                        |
    |  (ポーリング開始: 例 10秒間隔)          |
    |                                        |
    |-- GET .../status/{task_id} ----------> |
    |<-- 202 Accepted (処理中)               |
    |    { "progress_percent": 45 }          |
    |                                        |
    |-- GET .../status/{task_id} ----------> |
    |<-- 303 See Other (完了)                |
    |    Location: .../results/{task_id}     |
    |                                        |
    |-- GET .../results/{task_id} ---------> |
    |<-- 200 OK                              |
    |    { "results": [...] }                |
```

### HTTPステータスコード一覧

| フェーズ | エンドポイント | ステータスコード | 意味 |
|---------|--------------|----------------|------|
| 処理受付 | `POST .../batch_conversions` | `202 Accepted` | ジョブ受付成功、処理はバックグラウンドで継続 |
| 処理中 | `GET .../status/{task_id}` | `202 Accepted` | 処理継続中 |
| 処理完了 | `GET .../status/{task_id}` | `303 See Other` | 処理完了、結果URLへリダイレクト |
| 結果取得 | `GET .../results/{task_id}` | `200 OK` | 結果データの返却 |
| 処理失敗 | `GET .../status/{task_id}` | `200 OK` | 失敗ステータスをボディに含めて返却 |
| 認証エラー | 全エンドポイント | `401 Unauthorized` | 認証トークン不正・期限切れ |
| 権限エラー | 全エンドポイント | `403 Forbidden` | アクセス権限なし |
| リソース不存在 | `GET .../status/{task_id}` | `404 Not Found` | 指定した task_id が存在しない |

---

## 5. タイムアウトと完了通知

- **[MUST]** 適切なタイムアウト設定を行うこと（例: ジョブ有効期限 24 時間、結果保持期間 7 日間）
- **[MAY]** Webhook による完了通知を実装することができる

### Webhook 完了通知（オプション）

処理開始リクエスト時に `callback_url` を含めることで、完了時に通知を受け取ることができます。

```json
POST /v1/resident_records/batch_conversions

{
  "source_format": "xml",
  "target_format": "json",
  "record_ids": ["REC-00001", "REC-00002", "REC-00003"],
  "callback_url": "https://your-app.example.jp/webhooks/conversion-completed"
}
```

完了時に以下の形式でコールバックが送信されます：

```json
POST https://your-app.example.jp/webhooks/conversion-completed

{
  "task_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "completed",
  "result_url": "https://api.example.go.jp/v1/resident_records/batch_conversions/results/123e4567-e89b-12d3-a456-426614174000",
  "completed_at": "2026-04-12T10:03:15+09:00"
}
```

---

## 参考資料

- デジタル庁「APIテクニカルガイドブック（DS-464-2）」2024年9月30日版
  - 非同期API設計基準（処理時間1秒以上の場合）
  - URI設計規則（名詞・複数形・snake_case・バージョニング）
  - HTTPステータスコード利用規則（202/303の使い分け）
  - エラーレスポンスフォーマット
