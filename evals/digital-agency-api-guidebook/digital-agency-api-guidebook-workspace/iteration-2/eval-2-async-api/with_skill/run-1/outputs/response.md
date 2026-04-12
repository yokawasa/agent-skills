# 住民票一括データ変換処理 非同期API仕様

デジタル庁「APIテクニカルガイドブック（DS-464-2, 2024年9月30日版）」に基づく非同期API設計です。
処理時間が数分かかるため、非同期APIパターンを採用します（目安：処理時間 1秒以上 → 非同期API）。

---

## 1. 処理開始エンドポイント

### URI

```
POST /v1/residents/bulk_conversions
```

- サブドメイン: `api.example.go.jp`（api サブドメイン推奨）
- バージョン: `/v1/` を含める [SHOULD]
- リソース名: snake_case の複数形名詞 [SHOULD]

### HTTPメソッド

**POST** [SHOULD]

> 注意: ジョブ生成に POST を使用することは [SHOULD]（推奨）であり、[MUST]（必須）ではありません。

### リクエスト例

```http
POST /v1/residents/bulk_conversions
Host: api.example.go.jp
Content-Type: application/json
Authorization: Bearer {token}

{
  "source_format": "xml",
  "target_format": "json",
  "target_date": "2024-04-01",
  "municipality_code": "131016"
}
```

### レスポンス: 202 Accepted [MUST]

受付時は必ず **202 Accepted** を返し、**Location ヘッダー**にステータス確認URLを含めます。

```http
HTTP/1.1 202 Accepted
Location: /v1/residents/bulk_conversions/status/job-20240401-abc123
Content-Type: application/json

{
  "job_id": "job-20240401-abc123",
  "status": "accepted",
  "message": "住民票一括データ変換ジョブを受け付けました。",
  "status_url": "/v1/residents/bulk_conversions/status/job-20240401-abc123",
  "accepted_at": "2024-04-01T09:00:00+09:00"
}
```

---

## 2. ステータス確認エンドポイント

### URI

```
GET /v1/residents/bulk_conversions/status/{job_id}
```

### レスポンス パターン A: 処理中 → 202 Accepted [MUST]

処理がまだ完了していない場合は **202 Accepted** を返します。

```http
HTTP/1.1 202 Accepted
Content-Type: application/json

{
  "job_id": "job-20240401-abc123",
  "status": "in_progress",
  "message": "住民票データを変換中です。しばらくお待ちください。",
  "progress_percent": 45,
  "estimated_completion_at": "2024-04-01T09:05:00+09:00",
  "checked_at": "2024-04-01T09:02:15+09:00"
}
```

### レスポンス パターン B: 処理完了 → 303 See Other [MUST]

処理が完了したら **303 See Other** を返し、**Location ヘッダー**に結果取得URLを含めます。

```http
HTTP/1.1 303 See Other
Location: /v1/residents/bulk_conversions/results/job-20240401-abc123
Content-Type: application/json

{
  "job_id": "job-20240401-abc123",
  "status": "completed",
  "message": "住民票一括データ変換が完了しました。",
  "result_url": "/v1/residents/bulk_conversions/results/job-20240401-abc123",
  "completed_at": "2024-04-01T09:04:50+09:00"
}
```

### レスポンス パターン C: タイムアウト → エラー [MUST]

タイムアウト処理は必須です [MUST]。

```http
HTTP/1.1 408 Request Timeout
Content-Type: application/json

{
  "job_id": "job-20240401-abc123",
  "status": "timeout",
  "error": {
    "code": "JOB_TIMEOUT",
    "message": "処理がタイムアウトしました。再度お試しください。",
    "timed_out_at": "2024-04-01T09:30:00+09:00"
  }
}
```

---

## 3. 結果取得エンドポイント（303 リダイレクト先）

### URI

```
GET /v1/residents/bulk_conversions/results/{job_id}
```

### レスポンス: 200 OK

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "job_id": "job-20240401-abc123",
  "status": "completed",
  "result": {
    "total_records": 15240,
    "converted_records": 15240,
    "failed_records": 0,
    "output_file_url": "https://api.example.go.jp/v1/residents/bulk_conversions/files/job-20240401-abc123.json",
    "output_file_expires_at": "2024-04-08T09:04:50+09:00"
  },
  "completed_at": "2024-04-01T09:04:50+09:00"
}
```

---

## 4. 全体フロー（各フェーズのHTTPステータスコードまとめ）

```
クライアント                              APIサーバー
    |                                        |
    |-- POST /v1/residents/bulk_conversions -->|
    |<-- 202 Accepted + Location header ------|  [MUST]
    |                                        |
    |  (ポーリング or Webhook通知待ち)        |
    |                                        |
    |-- GET /v1/.../status/{job_id} -------->|
    |<-- 202 Accepted (処理中) --------------|  [MUST]
    |                                        |
    |-- GET /v1/.../status/{job_id} -------->|
    |<-- 303 See Other + Location header ----|  [MUST]
    |                                        |
    |-- GET /v1/.../results/{job_id} ------->|  (303のリダイレクト先)
    |<-- 200 OK (結果データ) -----------------|
    |                                        |
```

| フェーズ | エンドポイント | HTTPメソッド | ステータスコード | 必須度 |
|----------|---------------|-------------|-----------------|--------|
| ジョブ受付 | POST /v1/residents/bulk_conversions | POST | 202 Accepted + Location | [MUST] |
| 処理中確認 | GET /v1/residents/bulk_conversions/status/{job_id} | GET | 202 Accepted | [MUST] |
| 処理完了確認 | GET /v1/residents/bulk_conversions/status/{job_id} | GET | 303 See Other + Location | [MUST] |
| 結果取得 | GET /v1/residents/bulk_conversions/results/{job_id} | GET | 200 OK | - |
| タイムアウト | GET /v1/residents/bulk_conversions/status/{job_id} | GET | 408 Request Timeout | [MUST] |

---

## 5. Webhook通知（オプション）

Webhook による完了通知は **[MAY]**（任意）です。

```json
POST {callback_url}
Content-Type: application/json

{
  "event": "job.completed",
  "job_id": "job-20240401-abc123",
  "status": "completed",
  "result_url": "/v1/residents/bulk_conversions/results/job-20240401-abc123",
  "completed_at": "2024-04-01T09:04:50+09:00"
}
```

---

## 参考: ガイドライン準拠マッピング

| ガイドライン要件 | 本仕様での実装 | 必須度 |
|-----------------|---------------|--------|
| 処理時間 ≥ 1秒 → 非同期API | 数分かかる処理のため非同期採用 | - |
| 受付時 202 + Location ヘッダー | POST レスポンスで実装 | [MUST] |
| ステータス確認: 202（処理中） | GET /status/{job_id} で実装 | [MUST] |
| ステータス確認: 303（完了） | GET /status/{job_id} で実装 | [MUST] |
| タイムアウト処理 | 408 エラーレスポンスで実装 | [MUST] |
| Webhook通知 | コールバックURL経由で実装 | [MAY] |
| snake_case 複数形名詞 URI | bulk_conversions, residents | [SHOULD] |
| /v1/ バージョン含む | URI に /v1/ を含む | [SHOULD] |
| api サブドメイン | api.example.go.jp | [SHOULD] |
| ジョブ作成に POST | POST メソッドを使用 | [SHOULD] |
