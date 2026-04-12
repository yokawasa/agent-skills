# 行政手続きAPIにおける422バリデーションエラーレスポンス設計

## 政府標準（デジタル庁APIテクニカルガイドブック）に基づく設計方針

日本のデジタル庁「APIテクニカルガイドブック（DS-464-2, 2024年9月30日版）」では、エラーレスポンスについて以下の要件を定めています。

### 基本要件

- **[SHOULD]** エラーレスポンスは RFC 7807「Problem Details for HTTP APIs」形式に従うこと
- **[MUST]** `Content-Type` ヘッダーには `application/problem+json` を使用すること
- **[SHOULD]** 以下の5つのフィールドをすべて含めること

| フィールド | 説明 |
|---|---|
| `status` | HTTPステータスコード（数値） |
| `type` | エラー種別を示すURI（定義がない場合は `"about:blank"` を使用） |
| `title` | エラー名称（人が読める短い文字列） |
| `detail` | エラーの説明（APIユーザーが問題を理解できる十分な内容） |
| `instance` | エラーが発生したリソースのURI |

- **[MAY]** アプリケーション固有の拡張フィールドを追加できる

### バリデーションエラーのステータスコード

バリデーションエラーには HTTP **422 Unprocessable Entity** を使用します。

---

## 設計例：2つのバリデーションエラーが同時に発生する場合

### シナリオ

- 申請者氏名（`applicant_name`）が空
- 生年月日（`birth_date`）のフォーマット不正

### レスポンス設計

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json
Content-Language: ja
```

```json
{
  "type": "https://api.example.go.jp/probs/validation-error",
  "title": "入力値の検証に失敗しました。",
  "status": 422,
  "detail": "リクエストに含まれる1つ以上のフィールドが無効です。各フィールドのエラー内容を確認し、修正してから再送信してください。",
  "instance": "/applications/apply",
  "errors": [
    {
      "field": "applicant_name",
      "message": "申請者氏名は必須項目です。空のまま送信することはできません。"
    },
    {
      "field": "birth_date",
      "message": "生年月日のフォーマットが不正です。YYYY-MM-DD形式（例：1990-04-01）で入力してください。"
    }
  ]
}
```

---

## 各フィールドの設計根拠

| フィールド | 設定値 | 根拠 |
|---|---|---|
| `status` | `422` | バリデーションエラーに対応するHTTPステータスコード（ガイドブック規定） |
| `type` | `"https://api.example.go.jp/probs/validation-error"` | エラー種別を一意に識別するURI。自組織のAPIドメインで管理する。定義がない場合は `"about:blank"` でも可 |
| `title` | `"入力値の検証に失敗しました。"` | 人が読めるエラー名称。短く明確に記述 |
| `detail` | （上記参照） | APIユーザーが問題を理解し、次の行動を取れるよう十分な説明を記載 **[SHOULD]** |
| `instance` | `"/applications/apply"` | エラーが発生したエンドポイントのURI |
| `errors` | 配列（拡張フィールド） | RFC 7807 拡張フィールド **[MAY]**。複数バリデーションエラーを一括で返すために使用 |

---

## 複数バリデーションエラーへの対応

RFC 7807 の標準フィールドのみでは複数エラーを一度に伝えることが難しいため、**[MAY]** の拡張フィールドとして `errors` 配列を追加することを推奨します。

- `errors[].field`: エラーが発生したフィールド名（APIのリクエストパラメータ名と対応）
- `errors[].message`: そのフィールド固有のエラーメッセージ（ユーザーが修正内容を理解できる具体的な説明）

この設計により、APIクライアント（フロントエンドや他システム）はフィールド単位でエラーを処理し、ユーザーへの適切なフィードバックが可能になります。

---

## まとめ

政府標準に沿った422バリデーションエラーレスポンスの要点：

1. **[MUST]** `Content-Type: application/problem+json` を必ず付与する
2. **[SHOULD]** RFC 7807 の5フィールド（`status`, `type`, `title`, `detail`, `instance`）をすべて含める
3. **[MAY]** 拡張フィールド `errors` 配列で複数バリデーションエラーをフィールド単位に列挙する
4. `type` URIは自組織のAPIドメインで管理し、エラー種別ごとに一意なURIを割り当てる（URI先でエラーの詳細説明ページを提供することが望ましい）
