# 住民票API エンドポイント設計レビュー

**根拠:** デジタル庁「APIテクニカルガイドブック（DS-464-2, 2024年9月30日版）」

---

## 現行エンドポイントの問題点と改善案

### エンドポイント 1: `POST /api/v1/getResidentCard/{userId}`

#### 問題点

1. **[SHOULD] URIに動詞が含まれている**
   - `getResidentCard` は動詞句であり、URIに操作を含めてはならない。
   - ガイドブックでは「操作はHTTPメソッドで表現し、URIにはリソースを表す名詞を使うこと」と定めている。

2. **[SHOULD] HTTPメソッドがCRUDセマンティクスと不一致**
   - `POST` はリソースの「作成」を意味するが、この操作は住民票データの「取得（Read）」である。
   - 取得操作には `GET` を使用しなければならない。

3. **[SHOULD] camelCaseが使われている**
   - `getResidentCard` は camelCase 表記であり、ガイドブックでは snake_case を推奨している。

#### 改善案

```
GET /api/v1/resident_cards/{user_id}
```

- `resident_cards` : 名詞・複数形・snake_case
- `{user_id}` : パスパラメータも snake_case に統一
- `GET` : 読み取り操作に対応する正しいHTTPメソッド

---

### エンドポイント 2: `GET /api/v1/updateAddress`

#### 問題点

1. **[SHOULD] URIに動詞が含まれている**
   - `updateAddress` は動詞句であり、リソースではなく操作を表している。

2. **[SHOULD] HTTPメソッドがCRUDセマンティクスと不一致**
   - `GET` は読み取り操作を意味するが、`updateAddress` という名前は更新（Update）操作を示している。
   - 更新操作には `PUT`（全体更新）または `PATCH`（部分更新）を使用すること。

3. **[SHOULD] camelCaseが使われている**
   - `updateAddress` は camelCase 表記であり、snake_case に変更すること。

4. **[SHOULD] リソースの階層構造が不明確**
   - どのユーザーの住所を更新するのか、URIから判別できない。

#### 改善案

住所を全体置換する場合（PUT）:
```
PUT /api/v1/users/{user_id}/addresses/{address_id}
```

住所を部分更新する場合（PATCH）:
```
PATCH /api/v1/users/{user_id}/addresses/{address_id}
```

- `users/{user_id}/addresses` : ユーザーに紐づく住所リソースを階層で表現
- `PUT` / `PATCH` : 更新操作に対応する正しいHTTPメソッド

---

### エンドポイント 3: `DELETE /api/v1/userAddress/remove/{id}`

#### 問題点

1. **[SHOULD] URIに動詞が含まれている**
   - `remove` は動詞であり、削除操作を示している。
   - `DELETE` メソッドが既に削除を表すため、URI内の `remove` は冗長かつ規約違反。

2. **[SHOULD] リソース名がcamelCaseになっている**
   - `userAddress` は camelCase 表記であり、snake_case に変更すること。

3. **[SHOULD] リソース名が単数形になっている**
   - ガイドブックではリソース名に複数形を使用することを推奨している。
   - `userAddress` → `user_addresses` または階層表現 `users/{user_id}/addresses` が望ましい。

#### 改善案

```
DELETE /api/v1/users/{user_id}/addresses/{address_id}
```

- `users/{user_id}/addresses/{address_id}` : 名詞・複数形・snake_case・階層構造
- `DELETE` : 削除操作に対応するHTTPメソッドのみで操作を表現（URIに `remove` 不要）

---

## 修正前後の比較表

| # | 修正前 | 修正後 | 主な問題点 |
|---|--------|--------|-----------|
| 1 | `POST /api/v1/getResidentCard/{userId}` | `GET /api/v1/resident_cards/{user_id}` | 動詞URI、POSTでの取得、camelCase |
| 2 | `GET /api/v1/updateAddress` | `PUT /api/v1/users/{user_id}/addresses/{address_id}` | 動詞URI、GETでの更新、camelCase |
| 3 | `DELETE /api/v1/userAddress/remove/{id}` | `DELETE /api/v1/users/{user_id}/addresses/{address_id}` | URI内の動詞 `remove`、camelCase、単数形 |

---

## チェックリスト（デジタル庁ガイドブック準拠確認）

| チェック項目 | 修正前 | 修正後 |
|-------------|--------|--------|
| 名詞・複数形を使用しているか | NG（動詞・単数形混在） | OK |
| snake_case を使用しているか | NG（camelCase） | OK |
| バージョン番号（`/v1/` 等）があるか | OK (`/v1/`) | OK |
| `api` サブドメインまたはパスを使用しているか | OK (`/api/`) | OK |
| HTTPメソッドで操作を表現し、URIに動詞がないか | NG（動詞URI・メソッド不一致） | OK |

---

## まとめ

現行の3エンドポイントはいずれもデジタル庁ガイドブックの [SHOULD] 規定に違反しています。主な問題は以下の3点です：

1. **URIに動詞が含まれている** (`getResidentCard`, `updateAddress`, `remove`) — 操作はHTTPメソッドで表現し、URIはリソースを名詞で表すこと
2. **HTTPメソッドの誤用** — `POST` で取得、`GET` で更新という逆転した使い方は避けること
3. **命名規則の不統一** — camelCase ではなく snake_case を使用し、リソース名は複数形にすること

改善後のエンドポイントはいずれも「リソース指向アーキテクチャ」の原則に沿った設計となっており、デジタル庁ガイドブックの推奨する REST API 設計ルールに準拠しています。
