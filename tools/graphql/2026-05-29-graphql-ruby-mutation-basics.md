---
title: GraphQL Ruby の Mutation 設計まとめ（BaseMutation / RelayClassicMutation）
date: 2026-05-29
tags: [graphql, ruby, rails, mutation, relay]
---

## 概要

GraphQL Ruby gem における Mutation の構造を学んだまとめ。`BaseMutation` と `RelayClassicMutation` の関係、`argument_class` 等 4 つのクラス設定の意味、空の Base クラスを最初から用意する理由までを整理する。

## 1. Mutation の基本構造

### 例: NotifyEventApplicants

主催者が自分のイベントの申込者に一斉メールを送る Mutation。

```ruby
class NotifyEventApplicants < BaseMutation
  argument :event_id, ID, required: true
  argument :target_status, Types::EventApplicationStatusType, required: true
  argument :subject, String, required: true
  argument :body, String, required: true

  field :success, Boolean, null: false
  field :errors, [Types::ValidationErrorType], null: false
  field :enqueued_count, Integer, null: true

  def resolve(event_id:, target_status:, subject:, body:)
    # ① バリデーション → ② 認可スコープでイベント取得
    # → ③ 宛先ユーザーID取得 → ④ Solid Queue にジョブ投入
  end
end
```

### 設計の重要ポイント

| 観点 | 実装 |
|---|---|
| 認可 | `Event.where(organizer: user.organizer).find_by(id:)` でクエリスコープに含める |
| 非同期化 | Solid Queue 経由で投入。Mutation は即返す |
| プライバシー | Bcc 不使用、Job 側で個別送信 |
| 入力制限 | `SUBJECT_MAX_LENGTH = 100`, `BODY_MAX_LENGTH = 5_000`（SES 課金 & DB 負荷対策） |
| `enqueued_count` の意味 | 「投入件数」であり「送信完了数」ではない |

## 2. BaseMutation - すべての Mutation の基底

このプロジェクト内の全 Mutation が `BaseMutation` を継承している規約。

`base_mutation.rb`:

```ruby
module Mutations
  class BaseMutation < GraphQL::Schema::RelayClassicMutation
    argument_class Types::BaseArgument
    field_class Types::BaseField
    input_object_class Types::BaseInputObject
    object_class Types::BaseObject
  end
end
```

中身は **最小構成**（プロジェクトの方針として YAGNI を採用）。共通処理は重複が見えてから足す。

## 3. RelayClassicMutation - なぜ input 階層を作るか

GraphQL Ruby gem には 2 系統 の Mutation 基底があり、このプロジェクトは Relay 形式を採用。

| 形式 | クエリの形 |
|---|---|
| `RelayClassicMutation`（このプロジェクトが採用） | `mutation { foo(input: { eventId, ... }) { ... } }` |
| `GraphQL::Schema::Mutation` | `mutation { foo(eventId, ...) { ... } }` |

### このプロジェクトが Relay 形式を選んだ理由

| 理由 | 効果 |
|---|---|
| variables = input 型の 1 対 1 対応 | フォーム state をそのまま input に渡せる |
| 名前付き Input 型の再利用 | `NotifyEventApplicantsInput` を TypeScript で再利用可能 |
| スキーマ進化に強い | argument 追加で既存クエリが壊れない |
| 見た目が統一される | 全 Mutation が同じパターンになる |
| Apollo + Codegen との相性 | TypeScript 型が綺麗にまとまる |
| `clientMutationId` 自動付与 | 楽観的更新が書きやすい |

### Ruby 側は input 階層を意識しなくていい

```ruby
def resolve(event_id:, target_status:, ...)
  # gem が input を展開してキーワード引数として渡してくれる
end
```

スキーマとフロントには階層あり、Ruby サーバ側にはなし。

## 4. BaseMutation の 4 つのクラス設定

Mutation が **内部で自動生成する型** すべてに対応している。

### Mutation が生成する 2 つの「箱」と「中身」

```graphql
mutation {
  notifyEventApplicants(input: { ... }) {   # ← ① Input オブジェクト
    success, errors, enqueuedCount          # ← ② Payload オブジェクト
  }
}
```

| 設定 | 対応する役割 | 軸 |
|---|---|---|
| `argument_class` | 入力フィールド 1 個 1 個 | 入力 × 部品 |
| `field_class` | 出力フィールド 1 個 1 個 | 出力 × 部品 |
| `input_object_class` | 入力をまとめる箱（`XxxInput`） | 入力 × 箱 |
| `object_class` | 返り値をまとめる箱（`XxxPayload`） | 出力 × 箱 |

「1 個 vs まとめ」「入力 vs 出力」の 2 軸で 4 種類と覚える。

### Enum や Connection が含まれない理由

| 種類 | 理由 |
|---|---|
| Enum / Scalar / Interface / Union | Mutation が自動生成しない（開発者が手で定義する型） |
| Connection / Edge | `BaseObject` 側で既に設定済み（`object_class` 経由で間接的に効く） |

## 5. 空の Base クラスの意義

### Types ディレクトリの構成

```
packages/backend/app/graphql/types/
├── base_argument.rb        ← ほぼ空
├── base_field.rb           ← argument_class 設定のみ
├── base_input_object.rb    ← argument_class 設定のみ
├── base_object.rb          ← edge/connection/field 設定
├── base_enum.rb            ← 完全に空
├── base_scalar.rb          ← 空
├── base_interface.rb       ← 空
└── base_union.rb           ← 空
```

### なぜ空でも作っておくのか

YAGNI 原則の例外。理由は **「後付けコストが線形に膨らむタイプ」だから**。

#### シナリオ比較

**空 BaseEnum を作らなかった場合**:
- 後で全 Enum に共通処理を入れたい → 全 Enum ファイルを書き換え

**最初から空 BaseEnum を作っておいた場合**:
- 後で全 Enum に共通処理を入れたい → `BaseEnum` 1 ファイルに追記するだけ

### Rails の ApplicationController と同じ思想

```ruby
class ApplicationController < ActionController::Base
end  # 最初は空でも、後で current_user などを追加する場所として用意される
```

同じく `Types::BaseEnum` も「今は空、将来の拡張ポイントを予約」。

### YAGNI との折り合い

| 抽象化の種類 | 後付けコスト | YAGNI 適用 |
|---|---|---|
| 普通のヘルパーメソッド | 低い | 適用する（重複してから切り出す） |
| 基底クラス（継承元） | 高い | 適用しない（最初に挟んでおく） |

プロジェクト内の `validators/` に関する議論と同じで、継承元は別ルールで扱う。

## 6. 業界標準としての位置づけ

### `rails generate graphql:install` で自動生成される

```bash
rails g graphql:install
```

を実行すると、以下が gem インストーラによって自動生成される:

- `Mutations::BaseMutation`（4 行のクラス設定入り）
- `Types::Base*` 一式（ほぼ空）

つまりプロジェクト開発者が書いたわけではなく、規約として最初から揃っている。

### Rails + GraphQL Ruby のデファクト

| 観点 | 状態 |
|---|---|
| 自分で書く必要 | 基本なし（インストーラが書く） |
| 強制力 | 規約だが強制ではない |
| 他プロジェクトでも同じ？ | Shopify など大規模事例含めほぼ全部同じ |
| 公式推奨 | ガイドの基本構成として示されている |

## 7. 拡張する場合のパターン（参考）

将来 `BaseMutation` に共通処理を入れるとしたら:

### 認証ヘルパー

```ruby
def current_user
  context[:current_user]
end

def authenticate!
  raise GraphQL::ExecutionError, "ログインが必要です" unless current_user
end
```

### エラー整形ヘルパー

```ruby
def format_errors(record)
  record.errors.map { |e| { field: e.attribute.to_s, message: e.message } }
end
```

### トランザクション包み

```ruby
def resolve(**args)
  ActiveRecord::Base.transaction { super }
end
```

### Payload の null デフォルト変更

```ruby
null false
```

→ このプロジェクトは今のところ どれも採用していない（YAGNI）。

## まとめ: 全体の階層構造

```
GraphQL::Schema::RelayClassicMutation     ← gem 提供
  └─ Mutations::BaseMutation              ← 4 つの class 設定
       ├─ argument_class → Types::BaseArgument
       ├─ field_class    → Types::BaseField
       ├─ input_object_class → Types::BaseInputObject
       └─ object_class   → Types::BaseObject
                                ├─ field_class → Types::BaseField
                                ├─ edge_type_class → Types::BaseEdge
                                └─ connection_type_class → Types::BaseConnection

  └─ Mutations::NotifyEventApplicants     ← 個別 Mutation
  └─ Mutations::Login
  └─ Mutations::CreateEvent
  └─ ... (プロジェクト内の全 Mutation)
```

## ポイント

- すべての Mutation は `BaseMutation` 経由でプロジェクト基底に統一されている
- Relay 形式により `input:` 階層が増えるが、フォーム state ↔ 型の対応が綺麗になる
- 4 つの class 設定は「入力 × 出力」「部品 × 箱」の 2 軸で過不足なく揃っている
- 空の Base クラスは「今は空、将来の拡張ポイント」として規約に含まれている
- これらはこのプロジェクト固有ではなく、Rails + GraphQL Ruby のデファクト構成
