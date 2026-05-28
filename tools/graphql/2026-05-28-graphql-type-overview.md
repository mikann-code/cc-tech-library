---
title: GraphQL の type について — 学びまとめ
date: 2026-05-28
tags: [graphql, ruby, rails, type-system, whitelist]
---

## 概要

**API で使うカラムは type に必ず定義する。書かなければ公開されない（ホワイトリスト方式）ため、漏れにくい。**

GraphQL における `type` の位置づけ・種類・DB との関係・REST との設計思想の違いまでを横断的にまとめたもの。

## そもそも type とは

`app/graphql/types/` 配下のファイル群。
GraphQL でやり取りされる**データの「形」を Ruby クラスで宣言する場所**。

| | 役割 |
|---|---|
| resolver | 読み取り処理（どう取得するか） |
| mutation | 書き込み処理（どう作る／変えるか） |
| **type** | **入出力の構造定義（どんなデータが行き来するか）** |

## type の種類

| 種類 | 基底クラス | 用途 | 例 |
|---|---|---|---|
| Object 型 | `Types::BaseObject` | DB レコード等の複数フィールドを持つ型 | `EventType` |
| Enum 型 | `Types::BaseEnum` | 取りうる値が決まっている列挙 | `EventApplicationStatusType` |
| Input 型 | `Types::BaseInputObject` | Mutation 引数の複合型 | — |
| Base〇〇 系 | — | フレームワークの土台。基本触らない | `BaseEdge` 等 |

## description の役割

```ruby
description "イベント申込のステータス（一斉送信の宛先絞り込み用）"
value "ALL", value: "all", description: "全申込者（ステータス問わず）"
```

- 普通のコメント `#` と違い、GraphQL スキーマに焼き込まれる
- フロントの自動生成 TypeScript、GraphiQL、API ドキュメントに出る
- バリデーション機能は無し。あくまで説明文
- 弾く仕事は `value` の値定義そのものがやる

### なぜ Enum 型にだけ書くか

- Object 型のフィールド（`title`, `startAt`）は名前で意味が伝わる
- Enum の値（`APPLIED`, `REJECTED`, `NO_SHOW`）は名前だけでは業務上の区別がわからない
- 特に `ALL` のような擬似値は唯一の定義場所になる
- → 「値の名前 ≠ 値の意味」のギャップを埋めるのが description

## type と DB の関係

### DB のカラム ≠ API のカラム

- DB のカラム = 保存の都合で決まる
- API のカラム = 利用者に見せる都合で決まる
- 両者は別物。type は「API として公開する意思表示の場所」

### 4 パターン

| DB に存在 | type に記載 | 結果 |
|---|---|---|
| ○ | ○ | API で取得できる（通常パターン） |
| ○ | × | DB にあるが公開されない（非公開カラム） |
| × | ○ | DB に無いが計算値として公開（`applications_count` 等） |
| × | × | 何もない |

## 一般的な作業フロー

1. migration を作る ← DB 構造を変える
2. モデルにバリデーション等を追加 ← Ruby 側のルール
3. type に field を追加 ← GraphQL に公開する
4. resolver / mutation を直す ← 必要に応じて入出力を変える
5. yarn codegen ← フロントの TS 型を再生成
6. フロントが新しい field を使う

忘れがちなのは **3 と 5**。書かないとフロントから見えない。

## REST API との違い：ホワイトリスト方式

### REST：ブラックリスト方式（漏れる前提で隠す）

```ruby
render json: @user.as_json(except: [:password_digest])
# ↑ 「これを隠す」と書く方式。書き忘れたら漏れる。
```

→ デフォルトが全公開。新カラム追加時に事故が起きやすい。

### GraphQL：ホワイトリスト方式（出すものだけ書く）

```ruby
class UserType < Types::BaseObject
  field :email, String, null: false
  # password_digest を書かない → 自動的に非公開
end
```

→ デフォルトが全非公開。新カラム足しても type に書かない限り絶対漏れない。

## REST と GraphQL の比較表

| 観点 | REST | GraphQL |
|---|---|---|
| デフォルトの公開範囲 | 全公開（隠すのを書く） | 全非公開（出すのを書く） |
| 新カラム追加時の挙動 | 自動で漏れる可能性 | 絶対に漏れない |
| クライアントが取れるデータ | エンドポイントの返却内容 | クライアントがクエリで指定したものだけ |
| 型の存在 | 任意 | 必須（スキーマが本体） |
| 書く量 | 少ない | 多い（各層で型を書く） |

## ポイント

- 「API で使うカラムは type に定義しないといけない」= 書き忘れても漏れない = 明示しないと公開されない = **セキュアバイデフォルト**
- REST より書く量は増えるが、その分だけ事故が減る
- これは「保守的」というより 「明示主義（explicit）」「契約駆動（contract-first）」 な設計思想
- type は「API として公開する意思表示の場所」であり、DB のカラム構成とは独立して設計する

## 読むと理解が深まる type ファイル（優先順）

- `query_type.rb` — 全 GET API の目次
- `mutation_type.rb` — 全書き込み API の目次
- `event_type.rb` — Object 型の典型（DB カラム + 計算値 + リレーション）
- `event_application_status_type.rb` — Enum 型と description
- `events_type.rb` + `pager_type.rb` — 配列・複合レスポンス
- `organizer_event_application_type.rb` — 継承で権限ごとに見せ方を変える
- `validation_error_type.rb` — Mutation エラー返却の定型
