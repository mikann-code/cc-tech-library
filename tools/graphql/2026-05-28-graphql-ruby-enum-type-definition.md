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
