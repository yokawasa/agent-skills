# 住民票一括データ変換処理 — 非同期API設計仕様

> 一般的なREST APIベストプラクティスに基づく設計。処理に数分かかる長時間ジョブを想定。

---

## 設計方針

長時間処理（数分）には **非同期ジョブパターン** を採用する。

1. クライアントが処理開始リクエストを送る
2. サーバーはジョブを受け付け、`202 Accepted` と `jobId` を即時返却する
3. クライアントはポーリングでジョブステータスを確認する
4. 処理完了後、クライアントが結果を取得する

---

## 1. 処理開始エンドポイント

### リクエスト

| 項目 | 内容 |
|------|------|
| メソッド | `POST` |
| URI | `/v1/residents/bulk-conversions` |
| Content-Type | `application/json` |

```http
POST /v1/residents/bulk-conversions
Content-Type: application/json
Authorization: Bearer <token>
```

**リクエストボディ例:**

```json
{
  "sourceFormat": "csv",
  "targetFormat": "xml",
  "fileReference": "s3://bucket/input/residents-2024.csv",
  "options": {
    "encoding": "UTF-8",
    "validationLevel": "strict"
  }
}
```

### レスポンス: `202 Accepted`

処理をキューに登録したことを示す。結果はまだない。

```http
HTTP/1.1 202 Accepted
Content-Type: application/json
Location: /v1/residents/bulk-conversions/jobs/job-abc123
```

```json
{
  "jobId": "job-abc123",
  "status": "accepted",
  "message": "処理をキューに登録しました。",
  "links": {
    "status": "/v1/residents/bulk-conversions/jobs/job-abc123",
    "cancel": "/v1/residents/bulk-conversions/jobs/job-abc123"
  },
  "createdAt": "2026-04-12T09:00:00Z",
  "estimatedDurationSeconds": 180
}
```

**ポイント:**
- `Location` ヘッダーでステータス確認URIを明示する
- `jobId` をレスポンスボディにも含め、クライアントが保持できるようにする
- `estimatedDurationSeconds` は任意だが、ポーリング頻度の参考になる

---

## 2. ステータス確認エンドポイント

### リクエスト

| 項目 | 内容 |
|------|------|
| メソッド | `GET` |
| URI | `/v1/residents/bulk-conversions/jobs/{jobId}` |

```http
GET /v1/residents/bulk-conversions/jobs/job-abc123
Authorization: Bearer <token>
```

### レスポンスパターン

#### 2a. 処理中: `200 OK` + status = `processing`

```http
HTTP/1.1 200 OK
Content-Type: application/json
Retry-After: 30
```

```json
{
  "jobId": "job-abc123",
  "status": "processing",
  "progress": {
    "percentage": 45,
    "processedRecords": 4500,
    "totalRecords": 10000
  },
  "createdAt": "2026-04-12T09:00:00Z",
  "updatedAt": "2026-04-12T09:01:30Z",
  "links": {
    "self": "/v1/residents/bulk-conversions/jobs/job-abc123",
    "cancel": "/v1/residents/bulk-conversions/jobs/job-abc123"
  }
}
```

**ポイント:**
- `Retry-After` ヘッダーで次にポーリングするまでの推奨待機秒数を返す
- `progress` は任意だが、UX向上のために推奨

#### 2b. 処理完了: `200 OK` + status = `completed`

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "jobId": "job-abc123",
  "status": "completed",
  "progress": {
    "percentage": 100,
    "processedRecords": 10000,
    "totalRecords": 10000
  },
  "result": {
    "outputFileReference": "s3://bucket/output/residents-2024.xml",
    "downloadUrl": "/v1/residents/bulk-conversions/jobs/job-abc123/result",
    "expiresAt": "2026-04-13T09:03:00Z",
    "recordCount": 10000,
    "fileSizeBytes": 5242880
  },
  "createdAt": "2026-04-12T09:00:00Z",
  "updatedAt": "2026-04-12T09:03:00Z",
  "completedAt": "2026-04-12T09:03:00Z",
  "links": {
    "self": "/v1/residents/bulk-conversions/jobs/job-abc123",
    "result": "/v1/residents/bulk-conversions/jobs/job-abc123/result"
  }
}
```

#### 2c. 処理失敗: `200 OK` + status = `failed`

> ジョブ自体は存在するので `200 OK` を返す。エラー詳細はボディで表現する。

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "jobId": "job-abc123",
  "status": "failed",
  "error": {
    "code": "CONVERSION_ERROR",
    "message": "入力ファイルの形式が不正です。",
    "details": [
      {
        "line": 152,
        "field": "birthDate",
        "issue": "日付フォーマットが不正 (期待値: YYYY-MM-DD, 実際値: 04/12/2026)"
      }
    ]
  },
  "createdAt": "2026-04-12T09:00:00Z",
  "updatedAt": "2026-04-12T09:01:10Z",
  "failedAt": "2026-04-12T09:01:10Z"
}
```

#### 2d. ジョブが存在しない: `404 Not Found`

```http
HTTP/1.1 404 Not Found
Content-Type: application/json
```

```json
{
  "type": "https://example.go.jp/errors/not-found",
  "title": "Job Not Found",
  "status": 404,
  "detail": "指定されたジョブID 'job-xyz999' は存在しません。",
  "instance": "/v1/residents/bulk-conversions/jobs/job-xyz999"
}
```

---

## 3. 結果取得エンドポイント

### リクエスト

| 項目 | 内容 |
|------|------|
| メソッド | `GET` |
| URI | `/v1/residents/bulk-conversions/jobs/{jobId}/result` |

```http
GET /v1/residents/bulk-conversions/jobs/job-abc123/result
Authorization: Bearer <token>
```

### レスポンスパターン

#### 3a. 結果ファイルをダウンロード: `200 OK`

```http
HTTP/1.1 200 OK
Content-Type: application/xml
Content-Disposition: attachment; filename="residents-2024.xml"
Content-Length: 5242880
```

*(変換済みXMLファイルのバイナリストリーム)*

#### 3b. または署名付きダウンロードURLへのリダイレクト: `302 Found`

ファイルが大きい場合はオブジェクトストレージへリダイレクトする方式も有効。

```http
HTTP/1.1 302 Found
Location: https://storage.example.go.jp/output/residents-2024.xml?token=...&expires=1744628580
```

#### 3c. まだ処理中: `409 Conflict`

```http
HTTP/1.1 409 Conflict
Content-Type: application/json
```

```json
{
  "type": "https://example.go.jp/errors/job-not-completed",
  "title": "Job Not Completed",
  "status": 409,
  "detail": "ジョブはまだ処理中です。ステータス確認エンドポイントで完了を確認してから再度リクエストしてください。",
  "instance": "/v1/residents/bulk-conversions/jobs/job-abc123/result",
  "jobStatus": "processing"
}
```

#### 3d. 結果の有効期限切れ: `410 Gone`

```http
HTTP/1.1 410 Gone
Content-Type: application/json
```

```json
{
  "type": "https://example.go.jp/errors/result-expired",
  "title": "Result Expired",
  "status": 410,
  "detail": "処理結果の保持期限が切れています。再度処理を開始してください。",
  "instance": "/v1/residents/bulk-conversions/jobs/job-abc123/result",
  "expiredAt": "2026-04-13T09:03:00Z"
}
```

---

## 4. 全体フロー図

```
クライアント                         サーバー
    |                                    |
    |-- POST /bulk-conversions --------->|
    |                                    | (ジョブをキューに登録)
    |<-- 202 Accepted + jobId -----------|
    |    Location: /jobs/job-abc123      |
    |                                    |
    |  (30秒待機 ← Retry-After参照)      |  (バックグラウンド処理中...)
    |                                    |
    |-- GET /jobs/job-abc123 ----------->|
    |<-- 200 OK (status: processing) ----|
    |                                    |
    |  (30秒待機)                        |  (処理継続...)
    |                                    |
    |-- GET /jobs/job-abc123 ----------->|
    |<-- 200 OK (status: completed) -----|
    |                                    |
    |-- GET /jobs/job-abc123/result ---->|
    |<-- 200 OK (XMLファイル) -----------|
    |                                    |
```

---

## 5. HTTPステータスコード一覧

| フェーズ | エンドポイント | 状況 | ステータスコード |
|---------|--------------|------|----------------|
| 処理開始 | POST /bulk-conversions | 受付成功 | **202 Accepted** |
| 処理開始 | POST /bulk-conversions | バリデーションエラー | **400 Bad Request** |
| 処理開始 | POST /bulk-conversions | 認証エラー | **401 Unauthorized** |
| 処理開始 | POST /bulk-conversions | 権限エラー | **403 Forbidden** |
| ステータス確認 | GET /jobs/{jobId} | 処理中・完了・失敗 | **200 OK** |
| ステータス確認 | GET /jobs/{jobId} | ジョブ不存在 | **404 Not Found** |
| 結果取得 | GET /jobs/{jobId}/result | 取得成功 | **200 OK** |
| 結果取得 | GET /jobs/{jobId}/result | リダイレクト | **302 Found** |
| 結果取得 | GET /jobs/{jobId}/result | まだ処理中 | **409 Conflict** |
| 結果取得 | GET /jobs/{jobId}/result | 有効期限切れ | **410 Gone** |
| 結果取得 | GET /jobs/{jobId}/result | ジョブ不存在 | **404 Not Found** |

---

## 6. 設計上の補足・注意点

### ポーリング間隔の制御
- `Retry-After` レスポンスヘッダーを使い、サーバーがポーリング間隔を制御する
- クライアントはこれを無視して短い間隔でポーリングしてはいけない

### ジョブIDの設計
- 予測不可能なUUID等を使用する（連番は禁止）
- 例: `job-550e8400-e29b-41d4-a716-446655440000`

### 処理結果の保持期間
- 結果ファイルは一定期間（例: 24時間）後に自動削除
- `expiresAt` をレスポンスに含め、クライアントに明示する

### キャンセル操作（任意）
- `DELETE /v1/residents/bulk-conversions/jobs/{jobId}` でジョブをキャンセル可能
- レスポンス: `200 OK` (キャンセル成功) または `409 Conflict` (既に完了済み)

### 冪等性
- 同一リクエストの重複送信対策として、`Idempotency-Key` ヘッダーの使用を推奨
- サーバーは同じキーのリクエストを重複処理しない

```http
POST /v1/residents/bulk-conversions
Idempotency-Key: client-unique-key-20260412-001
```
