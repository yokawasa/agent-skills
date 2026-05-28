# 住民票API エンドポイント設計レビュー

デジタル庁「APIテクニカルガイドブック（DS-464-2, 2024年9月30日版）」に基づいてレビューします。

---

## 現行エンドポイントの問題点と改善案

### 1. `POST /api/v1/getResidentCard/{userId}`

#### 問題点

| # | 違反内容 | 規範強度 |
|---|----------|----------|
| 1 | URIパスに動詞 `getResidentCard` を含んでいる。操作はHTTPメソッドで表現すべき | [SHOULD] |
| 2 | `getResidentCard` がcamelCase。snake_caseを使うべき | [SHOULD] |
| 3 | リソースの取得操作に `POST` を使っている。取得には `GET` を使うべき | [SHOULD] |
| 4 | サブドメインが `api.` ではなく `/api/` パスプレフィックスになっている | [SHOULD] |

#### 改善案

```
GET https://api.example.go.jp/v1/resident_cards/{user_id}
```

- 動詞 `get` を削除し、リソース名の名詞（複数形）`resident_cards` に変更
- `userId` を snake_case の `user_id` に変更
- 取得操作なので HTTPメソッドを `GET` に変更
- サブドメイン `api.` を使用

---

### 2. `GET /api/v1/updateAddress`

#### 問題点

| # | 違反内容 | 規範強度 |
|---|----------|----------|
| 1 | URIパスに動詞 `updateAddress` を含んでいる。操作はHTTPメソッドで表現すべき | [SHOULD] |
| 2 | `updateAddress` がcamelCase。snake_caseを使うべき | [SHOULD] |
| 3 | 更新操作に `GET` を使っている。更新には `PUT`（全体置換）または `PATCH`（部分更新）を使うべき | [SHOULD] |
| 4 | リソースを特定するIDがURIに含まれていない（どのユーザーの住所か不明） | [SHOULD] |
| 5 | サブドメインが `api.` ではなく `/api/` パスプレフィックスになっている | [SHOULD] |

#### 改善案

```
PATCH https://api.example.go.jp/v1/users/{user_id}/addresses/{id}
```

または住所の全体置換の場合：

```
PUT https://api.example.go.jp/v1/users/{user_id}/addresses/{id}
```

- 動詞 `update` を削除し、リソース名の名詞（複数形）`addresses` に変更
- 部分更新なら `PATCH`、全体置換なら `PUT` を使用
- 対象リソースを特定するIDをURIに含める
- 親リソース `users/{user_id}` を階層的に表現
- サブドメイン `api.` を使用

---

### 3. `DELETE /api/v1/userAddress/remove/{id}`

#### 問題点

| # | 違反内容 | 規範強度 |
|---|----------|----------|
| 1 | URIパスに動詞 `remove` を含んでいる。削除操作はHTTPメソッド `DELETE` で表現すべき | [SHOULD] |
| 2 | `userAddress` がcamelCase。snake_caseを使うべき | [SHOULD] |
| 3 | リソース名 `userAddress` が単数形。複数形を使うべき | [SHOULD] |
| 4 | サブドメインが `api.` ではなく `/api/` パスプレフィックスになっている | [SHOULD] |

#### 改善案

```
DELETE https://api.example.go.jp/v1/users/{user_id}/addresses/{id}
```

- 動詞 `remove` を削除（`DELETE` メソッドが操作を表現する）
- `userAddress` を snake_case の複数形 `addresses` に変更
- 親リソース `users/{user_id}` を階層的に表現
- サブドメイン `api.` を使用

---

## 修正前後の一覧

| 現行エンドポイント | 改善後エンドポイント |
|-------------------|-------------------|
| `POST /api/v1/getResidentCard/{userId}` | `GET https://api.example.go.jp/v1/resident_cards/{user_id}` |
| `GET /api/v1/updateAddress` | `PATCH https://api.example.go.jp/v1/users/{user_id}/addresses/{id}` |
| `DELETE /api/v1/userAddress/remove/{id}` | `DELETE https://api.example.go.jp/v1/users/{user_id}/addresses/{id}` |

---

## 補足：規範強度について

本レビューで指摘した事項はすべて **[SHOULD]（推奨）** です。デジタル庁ガイドブック（DS-464-2）では、HTTPメソッドとCRUD操作のマッピングを含むURI設計規則は「推奨」として位置づけられており、**[MUST]（必須）ではありません**。

ただし、政府APIとしての相互運用性・可読性・保守性の観点から、これらの推奨事項への準拠を強くお勧めします。
