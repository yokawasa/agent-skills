# 住民票API エンドポイント設計レビュー（一般的なREST APIベストプラクティスに基づく）

## 現状のエンドポイント

```
POST /api/v1/getResidentCard/{userId}
GET  /api/v1/updateAddress
DELETE /api/v1/userAddress/remove/{id}
```

---

## 問題点と改善案

### 1. `POST /api/v1/getResidentCard/{userId}`

#### 問題点

- **HTTPメソッドの誤用**: `POST` は「リソースの新規作成」を意味するメソッドです。住民票情報の**取得**には `GET` を使うべきです。
- **URIにアクション動詞が含まれている**: `getResidentCard` という動詞をURIに含めるのはRESTの原則に反します。URIはリソース（名詞）を表すべきであり、操作（動詞）はHTTPメソッドで表現します。
- **リソースの命名**: `ResidentCard` はキャメルケースですが、URIはケバブケース（kebab-case）またはスネークケースで統一するのが一般的です。

#### 改善案

```
GET /api/v1/users/{userId}/resident-card
```

または、住民票をトップレベルリソースとして扱う場合：

```
GET /api/v1/resident-cards/{userId}
```

---

### 2. `GET /api/v1/updateAddress`

#### 問題点

- **HTTPメソッドの誤用**: `GET` は「安全（safe）かつ冪等（idempotent）」な操作、つまり**読み取り専用**に使うメソッドです。住所の**更新**には `PUT`（全体置換）または `PATCH`（部分更新）を使うべきです。
- **URIにアクション動詞が含まれている**: `updateAddress` という動詞をURIに含めるのはRESTの原則に反します。
- **リソースの特定が曖昧**: どのユーザーの住所を更新するのかがURIから判断できません。

#### 改善案

特定ユーザーの住所を全体置換する場合：

```
PUT /api/v1/users/{userId}/address
```

特定ユーザーの住所を部分更新する場合：

```
PATCH /api/v1/users/{userId}/address
```

---

### 3. `DELETE /api/v1/userAddress/remove/{id}`

#### 問題点

- **URIにアクション動詞が含まれている**: `remove` という動詞をURIに含めるのはRESTの原則に反します。削除の意味は `DELETE` メソッド自体が表現するため不要です。
- **リソースの命名が不統一**: `userAddress` はキャメルケースですが、他のエンドポイントとも命名規則を統一すべきです。また、`/userAddress/remove/{id}` という構造は冗長です。

#### 改善案

```
DELETE /api/v1/users/{userId}/address
```

または、住所に固有のIDがある場合（複数住所を管理する設計の場合）：

```
DELETE /api/v1/users/{userId}/addresses/{addressId}
```

---

## 改善後のエンドポイント一覧

| 操作 | 変更前 | 変更後 |
|---|---|---|
| 住民票の取得 | `POST /api/v1/getResidentCard/{userId}` | `GET /api/v1/users/{userId}/resident-card` |
| 住所の更新 | `GET /api/v1/updateAddress` | `PUT /api/v1/users/{userId}/address` または `PATCH /api/v1/users/{userId}/address` |
| 住所の削除 | `DELETE /api/v1/userAddress/remove/{id}` | `DELETE /api/v1/users/{userId}/address` |

---

## まとめ：一般的なREST API設計原則

1. **URIにはリソース（名詞）のみを使い、動詞は使わない**
   - 悪い例: `/getUser`, `/deleteItem`, `/updateAddress`
   - 良い例: `/users/{id}`, `/items/{id}`, `/users/{id}/address`

2. **HTTPメソッドで操作を表現する**
   - `GET`: 取得（読み取り専用、安全・冪等）
   - `POST`: 新規作成
   - `PUT`: リソースの全体置換（冪等）
   - `PATCH`: リソースの部分更新
   - `DELETE`: 削除（冪等）

3. **URIの命名はケバブケース（kebab-case）を推奨**
   - 例: `/resident-card`, `/user-address`

4. **リソースの階層構造を明確にする**
   - 親子関係: `/users/{userId}/addresses/{addressId}`
   - リソースの所有関係をURIで表現する

5. **コレクションリソースには複数形を使う**
   - 例: `/users`, `/addresses`, `/resident-cards`
