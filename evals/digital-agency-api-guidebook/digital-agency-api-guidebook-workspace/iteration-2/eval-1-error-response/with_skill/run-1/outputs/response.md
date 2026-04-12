# 422バリデーションエラーレスポンスの設計（政府標準準拠）

## 基本方針

日本のデジタル庁「APIテクニカルガイドブック（DS-464-2, 2024年9月30日版）」では、エラーレスポンスの形式として **RFC 7807「Problem Details for HTTP APIs」** に準拠することを **[SHOULD]**（推奨）としています。

バリデーションエラーには **HTTP 422 Unprocessable Entity** を使用します。

---

## RFC 7807 必須フィールド（すべて [SHOULD]）

| フィールド | 説明 |
|---|---|
| `status` | HTTPステータスコード |
| `type` | エラー種別を識別するURI（なければ `"about:blank"` を使用） |
| `title` | エラーの名称 |
| `detail` | エラーの説明 |
| `instance` | エラーが発生したリソースのURI |

---

## レスポンス設計例

### ヘッダー

```
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json  ← [SHOULD]（MUSTではない）
```

> **注意:** `Content-Type: application/problem+json` はガイドラインで **[SHOULD]**（推奨）であり、**[MUST]**（必須）ではありません。

---

### レスポンスボディ（単一エラーの基本形）

RFC 7807 の5フィールドのみを使用した基本形です。

```json
{
  "status": 422,
  "type": "https://example.go.jp/problems/validation-error",
  "title": "バリデーションエラー",
  "detail": "入力内容に誤りがあります。各フィールドを確認してください。",
  "instance": "/api/applications/submit"
}
```

---

### レスポンスボディ（複数バリデーションエラーを含む実践的パターン）

「申請者氏名（`applicant_name`）が空」と「生年月日（`birth_date`）のフォーマット不正」が同時に発生した場合：

```json
{
  "status": 422,
  "type": "https://example.go.jp/problems/validation-error",
  "title": "バリデーションエラー",
  "detail": "入力内容に誤りがあります。各フィールドを確認してください。",
  "instance": "/api/applications/submit",
  "errors": [
    {
      "field": "applicant_name",
      "message": "申請者氏名は必須項目です。空欄にはできません。"
    },
    {
      "field": "birth_date",
      "message": "生年月日のフォーマットが不正です。YYYY-MM-DD 形式で入力してください。"
    }
  ]
}
```

> **重要:** `errors` 配列による複数エラーの列挙は、**ガイドラインには規定なし・一般的プラクティス**です。RFC 7807 では拡張フィールドの追加が許可されており、実務上の利便性から広く採用されているパターンですが、政府標準として定められたものではありません。実装時はシステム要件や関係機関との合意に基づいて採否を判断してください。

---

## 設計のポイントまとめ

| 項目 | 規定 | 内容 |
|---|---|---|
| エラーレスポンス形式 | [SHOULD] | RFC 7807 に準拠 |
| `status` フィールド | [SHOULD] | HTTPステータスコードと一致させる |
| `type` フィールド | [SHOULD] | エラー種別URI（なければ `about:blank`） |
| `title` フィールド | [SHOULD] | エラー名称 |
| `detail` フィールド | [SHOULD] | エラー詳細説明 |
| `instance` フィールド | [SHOULD] | エラー発生URIパス |
| `Content-Type` | [SHOULD] | `application/problem+json`（MUSTではない） |
| `errors` 配列拡張 | ガイドラインには規定なし | 一般的プラクティスとして利用可能 |
| HTTPステータス | — | 422 Unprocessable Entity を使用 |

---

## 参考規格

- RFC 7807: Problem Details for HTTP APIs
- デジタル庁「APIテクニカルガイドブック」DS-464-2（2024年9月30日版）
