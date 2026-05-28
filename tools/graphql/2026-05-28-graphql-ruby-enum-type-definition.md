---
title: graphql-ruby の Enum 型定義の読み解き（EventApplicationStatusType）
date: 2026-05-28
tags: [graphql, ruby, rails, enum, graphql-ruby]
---

## 概要

GraphQL の Enum 型 `EventApplicationStatusType` を題材に、graphql-ruby での Enum 型定義パターンを読み解く。
DB の enum と API 用 Enum を分離しつつ、モデル enum から自動展開して二重メンテを排除する設計。

## `Types::` 配下のファイルとは

GraphQL では API でやり取りされる**すべてのデータに型を定義する**必要がある。
graphql-ruby ライブラリでは、それらの型定義を `app/graphql/types/` ディレクトリに置く規約。

| 種類 | 基底クラス | 用途 | 例 |
|---|---|---|---|
| Object 型 | `Types::BaseObject` | DB レコードなど複数フィールドを持つオブジェクト | `EventType` |
| **Enum 型** | **`Types::BaseEnum`** | **取りうる値が決まっている列挙** | **`EventApplicationStatusType`** |
| Input 型 | `Types::BaseInputObject` | Mutation 引数として複数フィールドをまとめて渡す | `AddressInputType` |
| Scalar 型 | `Types::BaseScalar` | カスタム原始型（日付・URL 等） | — |

→ `event_application_status_type.rb` は **GraphQL の Enum 型を1つ宣言しているファイル**。

## サンプルコード

### 定義されるスキーマ

GraphQL スキーマ上では以下のように展開される：

```graphql
enum EventApplicationStatus {
  ALL
  APPLIED
  CANCELLED
  REJECTED
  ATTENDED
  NO_SHOW
}
```

### 使われる場所

`NotifyEventApplicants` mutation の `target_status` 引数の型として参照される：

```ruby
argument :target_status, Types::EventApplicationStatusType, required: true
```

→ フロントが不正な値を送ると GraphQL 層が自動で弾く（バリデーション不要）。

### クラス宣言

```ruby
class EventApplicationStatusType < Types::BaseEnum
```

- `Types::BaseEnum` 継承で Enum 型と宣言
- クラス名末尾の `Type` は自動で削られ、スキーマ上は `EventApplicationStatus` になる

### description

```ruby
description "イベント申込のステータス（一斉送信の宛先絞り込み用。ALL は全件指定）"
```

- スキーマに埋め込まれるドキュメント文字列
- GraphiQL 等のスキーマブラウザでホバー時に表示
- スキーマがそのまま API ドキュメントになる

### ALL の手動追加

```ruby
value "ALL", value: "all", description: "全申込者（ステータス問わず）"
```

| 引数 | 役割 |
|---|---|
| 第1引数 `"ALL"` | スキーマ上の名前（フロントが書く値） |
| `value: "all"` | Ruby 側に渡る実際の値（resolver で受け取る文字列） |
| `description:` | この値の説明 |

**なぜ ALL だけ手書きか**:
DB の enum には存在しない、API でしか使わない擬似値だから。
「絞り込まない」を表す UI 上の便宜的な選択肢であり、保存される状態ではない。

### モデル enum からの自動展開

```ruby
EventApplication.statuses.each_key do |key|
  value key.upcase, value: key, description: key
end
```

- `EventApplication.statuses` は Rails が `enum :status` から生成するハッシュ
- ループで `APPLIED`, `CANCELLED`, `REJECTED`, `ATTENDED`, `NO_SHOW` を自動登録
- `key.upcase` で GraphQL の命名慣習（大文字スネークケース）に変換
- `value: key` で Ruby 側は DB enum 値そのまま受け取れる

## ポイント

- **Single Source of Truth**: モデルの `enum :status` がマスタ。GraphQL Enum は自動追従する。`no_show` をモデルに追加した瞬間、この型にも `NO_SHOW` が自動で現れる。型ファイルを書き換える必要なし。
- **責務の分離**: 「DB に保存する状態」と「API のフィルタパラメータ」は別物として型を分けている。`ALL` を DB enum に混ぜると `EventApplication.create(status: :all)` が書けてしまい責務がブレるため。

| | DB enum | GraphQL Enum |
|---|---|---|
| `applied` 等 | ある | ある |
| `ALL` | なし | ある |

- **ALL は API 専用の擬似値**として手動追加、残りはモデル enum から自動展開して二重メンテを排除
- 型を切ることでフロントの不正値を GraphQL 層が自動拒否
- `description` で API ドキュメントを兼ねる

## 補足：なぜ Enum 型にだけ description が書かれているのか

> 「値の名前」≠「値の意味」 になるのが Enum。
> そのギャップを埋めるのが description。

「なぜ Enum 型にだけ description が書かれているのか」を、より構造的に掘り下げる。

### 結論：Enum は「名前だけでは意味が決まらない」唯一の型だから

GraphQL の型のうち、**Enum 型だけが「値（識別子）と意味が 1:1 で結びつかない」**性質を持つ。
他の型はフィールド名や引数名で意味が自己説明できる。だから description を書く必要がない。

### 型ごとの「名前の説明力」を比較

#### Object 型 — 名前で完結する

```ruby
class EventType < Types::BaseObject
  field :title, String, null: false
  field :start_at, GraphQL::Types::ISO8601DateTime, null: false
  field :capacity, Integer, null: true
end
```

→ `title` ／ `start_at` ／ `capacity` — フィールド名そのものが意味を表している。
→ description を書いても「タイトル」「開始日時」「定員」と書くだけになり、冗長。

#### Mutation の引数 — 名前で大体伝わる

```ruby
argument :event_id, ID, required: true
argument :subject, String, required: true
```

→ `event_id` は ID、`subject` は件名、と名前から自明。
→ 制約（長さ上限など）を書きたいときだけ description が活きる。

#### Enum 型 — 名前だけでは意味が決まらない

```graphql
enum EventApplicationStatus {
  APPLIED      # ← これ、誰が「申し込んだ」状態？キャンセル可能？
  CANCELLED    # ← 誰がキャンセル？参加者？主催者？
  REJECTED     # ← CANCELLED と何が違う？
  ATTENDED     # ← 自動判定？それとも主催者が手動マーク？
  NO_SHOW      # ← REJECTED や CANCELLED とどう区別する？
  ALL          # ← そもそも DB に存在しない値、なぜここに？
}
```

→ 値の識別子だけからは、業務上の意味が読み取れない。
→ ここに description がないと、フロント開発者は「コード読みに行く／バックエンドに聞く」しかない。

### なぜ Enum がそうなるか — 構造的な理由

Enum は「取りうる値の集合」を宣言する型。値そのものに業務上の意味（ドメインの定義）が乗っている：

| 型 | 説明したいもの | 説明の必要性 |
|---|---|---|
| Object 型のフィールド | 「これは title だ」 | 名前で伝わる |
| Enum 型の値 | 「APPLIED とは何を意味するか」（業務ルール） | 名前では伝わらない |

Enum は識別子（`APPLIED`）の裏側に**ドメインルール（誰がいつどう変更できるか）**が隠れている。
これは型ファイルだけ読んでも分からない情報なので、description で表面化させる必要がある。

### このプロジェクトでさらに必要性が高い理由

#### ① ステータスの種類が業務固有の区別を含む

特にこの 2 つは説明なしでは絶対に分からない：

```ruby
# rejected:  主催者都合によるキャンセル済み
# cancelled: 参加者自身によるキャンセル済み
```

→ 「PJ オーナー要望で参加者キャンセルと区別」という業務要件が背景にある（`event_application.rb:10`）。
→ description で出さないと、フロントで「どっちを表示する／どう色分けする」が決められない。

#### ② ALL のような擬似値が存在

```ruby
value "ALL", value: "all", description: "全申込者（ステータス問わず）"
```

→ DB enum に存在しないので、コードを追っても「ALL とは何か」が他のどこにも書かれていない。
→ description が唯一の定義の場所になる。書かないと存在意義そのものが不明になる。

#### ③ Enum はフロントの UI ラベルに直結する

```tsx
<MenuItem value={EventApplicationStatus.NoShow}>無断欠席</MenuItem>
```

→ フロント開発者は「NoShow に対する日本語ラベルは何にすべきか」を決める必要がある。
→ description が「無断欠席（主催者がマークした状態）」のように仕様の根拠になる。
→ Object 型のフィールドは（title → 「タイトル」のように）この種の意思決定が発生しない。

### 逆に Object 型に description を書かない理由

「書いたほうが良いのでは？」と思うかもしれないが、書きすぎは逆効果：

```ruby
# ❌ 冗長で価値が低い
field :title, String, null: false, description: "タイトル"
field :capacity, Integer, null: true, description: "定員"
```

→ 名前と同じことを繰り返しているだけで、スキーマがノイズで埋まる。
→ 本当に説明が必要な箇所（Enum や特殊な引数）が埋もれて目立たなくなる。

「書く価値が高いところにだけ書く」のが、結果的にドキュメント全体の信号対雑音比を上げることになる。

### まとめ

| 観点 | Object 型 | Enum 型 |
|---|---|---|
| 値の意味が名前から推測できる？ | ○ できる（title = タイトル） | × できない（APPLIED = ？） |
| ドメインルールを内包する？ | フィールドは内包しない | 値が内包する（誰がいつ遷移可能か） |
| フロント UI のラベル設計に直結する？ | しない | 直結する（選択肢ラベル） |
| description のコスパ | 低（冗長） | 高（必須レベル） |

→ 「Enum 型は値が不透明で、ドメインの意味を持ち、UI に直結するから、description のコスパが圧倒的に高い」。だからこのプロジェクトでは Enum 型にだけ description が書かれている、というのが本質。
