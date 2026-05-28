# § 4 API開発の進め方

出典: デジタル庁「APIテクニカルガイドブック」§ 4

---

## 1. API提供ポータルの基本機能

API利用者の利便性向上のため、以下の機能を備えたポータルを整備すること [SHOULD]。

| 機能カテゴリ | 内容 |
|------------|------|
| ポータルの基本機能 | APIカタログ、ドキュメント、仕様書の公開 |
| ユーザー管理とアクセス制御 | APIキー発行・管理、利用者登録機能 |
| サポートと情報共有 | 問い合わせ窓口、FAQ、フォーラム |
| 使用状況と分析 | API利用統計・アクセスログの提供 |
| 更新管理と通知 | バージョンアップ・廃止・変更の告知 |
| セキュリティとコンプライアンス | 利用規約、セキュリティポリシーの公開 |
| フィードバックと改善 | 利用者からの意見収集・反映の仕組み |

---

## 2. OpenAPI（Swagger）を活用したAPI開発 [SHOULD]

### OpenAPI Specificationとは
REST APIの記述・共有のためのオープン規格（元Swagger）。YAMLまたはJSONで記述する。

### OpenAPI文書の基本構造

```yaml
openapi: "3.0.0"
info:
  title: "書籍情報API"
  version: "1.0.0"
  description: "市立図書館の書籍情報を提供するAPI"
servers:
  - url: "https://api.example.go.jp/v1"
paths:
  /books:
    get:
      summary: "書籍一覧の取得"
      parameters:
        - name: limit
          in: query
          schema:
            type: integer
            maximum: 100
        - name: offset
          in: query
          schema:
            type: integer
      responses:
        "200":
          description: "正常取得"
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/BookList"
        "400":
          description: "リクエストエラー"
components:
  schemas:
    BookList:
      type: object
      properties:
        metadata:
          $ref: "#/components/schemas/Metadata"
        results:
          type: array
          items:
            $ref: "#/components/schemas/Book"
```

### 活用メリット
- **仕様とドキュメントの一元管理**（仕様が即ドキュメントになる）
- **自動コード生成**（クライアントSDK・サーバースタブ）
- **Swagger UI等による対話型ドキュメント**の自動生成
- チーム間・府省間での仕様共有が容易

---

## 3. ドキュメントの公開 [SHOULD]

### 公開すべき「API概要情報」の例

| 項目 | 内容例 |
|------|--------|
| API名称 | 書籍情報API |
| バージョン | v1 |
| 提供組織 | ○○省 |
| 概要 | 市立図書館の蔵書情報を提供するREST API |
| ベースURL | `https://api.example.go.jp/v1` |
| 認証方式 | APIキー（Authorizationヘッダー） |
| レスポンス形式 | JSON |
| 利用規約URL | https://example.go.jp/terms |
| 問い合わせ先 | api-support@example.go.jp |
| 変更履歴 | バージョンごとの変更内容 |

### ドキュメントに含める内容
- エンドポイント一覧（URI・メソッド・概要）
- リクエストパラメータの説明
- レスポンスの説明（メタデータ・データ項目）
- エラーコード一覧
- 使用例（リクエスト・レスポンスのサンプル）
- 入力規約（日時フォーマット等）

---

## 4. テストフォーム及びテスト環境 [SHOULD]

- **テストフォーム**: API利用者が実際に試せる対話型ドキュメント（Swagger UI等）を提供する
- **テスト環境（サンドボックス）**: 本番データに影響を与えずにAPIを試せる環境を提供する

---

## 5. サンプルプログラムの提供 [SHOULD]

- 主要言語（Python、JavaScript、Java等）でのAPI呼び出しサンプルコードを提供する
- サンプルデータ（レスポンス例）もあわせて提供する

---

## 6. 開発者コミュニティ [MAY]

- GitHubやフォーラム等を通じてAPI利用者コミュニティを形成し、情報共有・フィードバック収集を促進する

---

## 7. FAQ [SHOULD]

- よくある質問とその回答をFAQとして整備し、サポートコストを削減する
- FAQは定期的に更新し、利用者からの問い合わせ内容を反映する
