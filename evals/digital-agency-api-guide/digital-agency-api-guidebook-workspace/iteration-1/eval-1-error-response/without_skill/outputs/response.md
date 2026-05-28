# 行政手続きAPIにおける422バリデーションエラーのレスポンス設計

## 基本方針

REST API における 422 Unprocessable Entity エラーのレスポンスボディ設計には、**RFC 7807 (Problem Details for HTTP APIs)** が広く参照される標準仕様です。RFC 7807 は IETF が定めたもので、政府・公共機関のAPI設計でも採用が推奨されるケースが増えています。

---

## RFC 7807 に基づくレスポンス設計

### Content-Type

```
Content-Type: application/problem+json
```

RFC 7807 では、Problem Details レスポンスの Content-Type として `application/problem+json` を使用することを定めています。

### 標準フィールド

| フィールド | 型 | 説明 |
|---|---|---|
| `type` | string (URI) | エラーの種別を示す URI。人間可読なドキュメントにリンクすることが推奨 |
| `title` | string | エラー種別の短い人間可読な説明（`type` ごとに固定値） |
| `status` | integer | HTTP ステータスコード（422） |
| `detail` | string | このリクエスト固有の詳細説明 |
| `instance` | string (URI) | このエラー発生の固有 URI（任意） |

### バリデーションエラー向け拡張フィールド

RFC 7807 は拡張フィールドの追加を認めています。バリデーションエラーでは、**複数フィールドのエラーをまとめて返す**ために `errors` 配列（または `invalid_params` 等）を追加するのが一般的なベストプラクティスです。

---

## 具体的なレスポンス例

### シナリオ
- `applicant_name`（申請者氏名）が空
- `birth_date`（生年月日）のフォーマットが不正（`YYYY-MM-DD` 形式でない）

### HTTPレスポンス

```
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json
```

```json
{
  "type": "https://api.example.go.jp/problems/validation-error",
  "title": "入力値のバリデーションエラーが発生しました",
  "status": 422,
  "detail": "リクエストボディに1件以上のバリデーションエラーがあります。各フィールドのエラー内容を確認してください。",
  "instance": "/applications/requests/20240412-0001",
  "errors": [
    {
      "field": "applicant_name",
      "code": "REQUIRED",
      "message": "申請者氏名は必須入力です。"
    },
    {
      "field": "birth_date",
      "code": "INVALID_FORMAT",
      "message": "生年月日はYYYY-MM-DD形式で入力してください。"
    }
  ]
}
```

---

## 設計のポイント解説

### 1. `type` フィールドは安定したURIを使う
- `type` は「このエラーの種類を一意に識別するURI」であり、実際にアクセス可能なドキュメントへのリンクにすることが推奨されます。
- 同じ種類のエラーには常に同じ `type` を使います（例: バリデーションエラーなら常に `validation-error`）。

### 2. `title` は `type` ごとに固定にする
- `title` はリクエストごとに変えず、`type` に対して一定の文言にします。
- リクエスト固有の情報は `detail` に含めます。

### 3. 複数エラーをまとめて返す
- バリデーションエラーは複数発生することが多いため、1回のレスポンスで**全てのエラー**を返す設計が利用者にとって親切です（「一つ直したらまた別のエラー」という繰り返しを防ぐ）。
- `errors` 配列に各フィールドのエラーをまとめます。

### 4. `errors` 配列の各要素
- `field`: どのフィールドでエラーが起きたか（リクエストボディのキー名）
- `code`: マシン可読なエラーコード（`REQUIRED`, `INVALID_FORMAT` 等）
- `message`: 人間可読な説明（日本語対応も可）

### 5. `instance` フィールド（任意）
- この特定のエラー発生を識別する URI を入れます。
- ログとの突合やサポート問い合わせ時のトレーサビリティに有効です。

---

## エラーコードの例

| コード | 意味 |
|---|---|
| `REQUIRED` | 必須フィールドが未入力・null |
| `INVALID_FORMAT` | フォーマット不正（日付形式、メール形式等） |
| `INVALID_VALUE` | 値が許可された範囲・選択肢に含まれない |
| `TOO_LONG` | 最大文字数超過 |
| `TOO_SHORT` | 最小文字数未満 |
| `DUPLICATE` | 重複した値（一意制約違反） |

---

## まとめ

| 項目 | 推奨値・方針 |
|---|---|
| ステータスコード | `422 Unprocessable Entity` |
| Content-Type | `application/problem+json` |
| 標準仕様 | RFC 7807 (Problem Details for HTTP APIs) |
| 複数エラー | `errors` 配列で一度に全て返す |
| エラー識別 | `code` フィールドでマシン可読コードを提供 |
| 人間可読情報 | `message` フィールドで日本語説明を提供 |

RFC 7807 ベースの設計は、仕様が公開標準であること、多くのフレームワーク・クライアントライブラリがサポートしていること、拡張性があることから、行政手続きAPIを含む公共サービスAPIの設計に適しています。
