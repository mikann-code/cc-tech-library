---
title: Rails の enum 完全ガイド（初学者向け・実務編）
date: 2026-06-09
tags: [rails, ruby, enum, active-record, graphql]
source: インターン学習メモ
---

## 概要

あるイベント管理アプリ（このプロジェクト）で実際に使われている enum を題材に、「enum とは何か」から「実務での使い方・落とし穴」までをまとめる。

対象モデル: **User / Event / EventApplication**

このプロジェクトの方針は **「ハッシュ記法で文字列保存」**（`enum :status, { active: "active" }`）。値の追加・並び替えで既存データが壊れないことを重視している。

## 目次

1. enum とは何か（概念）
2. なぜ enum を使うのか
3. 基本の書き方
4. enum が自動で作ってくれるメソッド
5. 【最重要】整数で保存するか、文字列で保存するか
6. よく使うオプション
7. このプロジェクトの実例
8. scope と組み合わせる（実務パターン）
9. GraphQL / フロントとの連携
10. よくある落とし穴
11. 新しい enum を追加する手順
12. チートシート

---

## 1. enum とは何か（概念）

enum（イーナム / 列挙型）とは、「取りうる値があらかじめ決まっている項目」を表現するための仕組み。

身近な例で考えると、こういうものが enum 向き。

- 注文の状態 → 未払い / 支払済み / 発送済み / キャンセル
- 会員ランク → ブロンズ / シルバー / ゴールド
- このプロジェクトのイベント状態 → 下書き / 予約公開 / 公開中 / 中止

「自由な文字列」ではなく「決められた選択肢の中の1つ」を持つカラムが enum の出番。

## 2. なぜ enum を使うのか

enum を使わずに、ただの文字列カラムで状態を管理することもできる。しかし、それには問題がある。

### ❌ enum を使わない場合（生の文字列）

```ruby
# status は普通の string カラム
user.status = "suspended"
user.save

# 状態をチェックするたびに文字列比較
if user.status == "suspended"
  # ...
end
```

この書き方の弱点:

- **タイプミスに気づけない** … `"suspeded"`（typo）と書いてもエラーにならず、バグが静かに紛れ込む
- **コードが冗長** … `user.status == "suspended"` を何度も書くことになる
- **どんな値があるのか分からない** … モデルを見ても取りうる値が一覧化されていない

### ✅ enum を使う場合

```ruby
enum :status, { active: "active", suspended: "suspended" }
```

たったこれだけで、

- `user.suspended?` という読みやすい述語メソッドが使える
- `User.suspended` で絞り込みスコープが使える
- モデルを見れば取りうる値が一目でわかる
- 定義にない値を入れようとするとエラーになる（安全）

つまり enum は「状態管理を安全に・読みやすくする道具」。

## 3. 基本の書き方

Rails 7 以降の書き方（このプロジェクトのスタイル）はこう。

```ruby
class User < ApplicationRecord
  enum :status, {
    active: "active",       # キー（コードで使う名前）: 値（DBに保存される文字列）
    suspended: "suspended"
  }
end
```

ポイントは `{ キー => 値 }` の形になっていること。

- **キー（`active`）** … Ruby コードの中で使う名前。`user.active?` のメソッド名になる
- **値（`"active"`）** … 実際に DB に保存される値

💡 このプロジェクトでは「キーと値を同じ文字列」にしている。理由はセクション5。

## 4. enum が自動で作ってくれるメソッド

enum を1行書くだけで、Rails がたくさんのメソッドを自動生成してくれる。これが enum の最大の便利さ。

User の `status` enum を例にすると、以下がすべて自動で使える。

### ① 述語メソッド（今どの状態か？を true/false で返す）

```ruby
user.active?      # => true なら active
user.suspended?   # => true なら suspended
```

### ② bang（!）メソッド（状態を変えて即DB保存）

```ruby
user.suspended!   # status を "suspended" に変更して save まで実行
user.active!
```

`!` 付きは「変更 + 保存」を一気に行う。保存まで済むので便利だが、バリデーションに失敗すると例外が出る点に注意。

### ③ setter（値をセットするだけ。保存はしない）

```ruby
user.status = :suspended   # シンボルでセット
user.status = "suspended"  # 文字列でもOK
user.save                  # 保存は別途必要
```

### ④ スコープ（条件で絞り込む）

```ruby
User.active       # status が active のユーザーだけ取得
User.suspended    # status が suspended のユーザーだけ取得
```

`where(status: "active")` と書く代わりに `User.active` と書ける。

### ⑤ 全選択肢の一覧を取得

```ruby
User.statuses
# => { "active" => "active", "suspended" => "suspended" }
```

セレクトボックスの選択肢を作るときなどに便利。

## 5. 【最重要】整数で保存するか、文字列で保存するか

ここが enum で最もハマりやすく、最も重要なポイント。

### enum の2つの書き方

```ruby
# 書き方A: 配列で書く → DBには「整数」で保存される
enum :status, [:active, :suspended]
# active → 0, suspended → 1 として保存される

# 書き方B: ハッシュで書く → DBには「文字列」で保存される（このプロジェクトの方式）
enum :status, { active: "active", suspended: "suspended" }
# active → "active", suspended → "suspended" として保存される
```

### なぜこのプロジェクトは「文字列保存」なのか？

整数保存（書き方A）には、**値を追加・変更するときに事故が起きやすい**という弱点がある。

たとえば最初こう定義したとする。

```ruby
enum :status, [:active, :suspended]
# active=0, suspended=1
```

DB には `0` や `1` が保存されている。後から「途中に新しい状態を挿入」するとどうなるか。

```ruby
enum :status, [:active, :pending, :suspended]  # 途中に pending を追加
# active=0, pending=1, suspended=2  ← suspended が 1 から 2 にズレた！
```

すでに保存されている `1`（元 suspended）が、突然 pending として解釈されてしまう。**これはデータ破壊につながる重大なバグ。**

一方、文字列保存（書き方B）なら、DB に `"suspended"` という文字列がそのまま入っているので、定義の順番を変えても・途中に値を足しても既存データは一切ズレない。

```ruby
# 途中に pending を足しても、DB の "suspended" は "suspended" のまま安全
enum :status, {
  active: "active",
  pending: "pending",     # 追加してもOK
  suspended: "suspended"
}
```

### このプロジェクトのルール

> enum は必ずハッシュ `{ キー => 値 }` で書き、文字列で保存する。
> カラムも `t.string` で定義する（`t.integer` ではない）。

User・Event・EventApplication すべてこの方式で統一されている。各モデルのコメントにも「DB には文字列で保存」と明記されている。

| | 整数保存（配列） | 文字列保存（ハッシュ）※採用 |
| --- | --- | --- |
| 書き方 | `enum :status, [:a, :b]` | `enum :status, { a: "a", b: "b" }` |
| DBの値 | `0`, `1` | `"a"`, `"b"` |
| カラム型 | `t.integer` | `t.string` |
| 値の追加・並び替え | 危険（ズレる） | 安全 |
| DBを直接見たとき | 意味が分からない（数字） | 意味が分かる（文字列） |
| 保存サイズ | 小さい | やや大きい |

文字列の方が保存サイズはわずかに大きくなるが、安全性と可読性のメリットが上回るため、このプロジェクトでは文字列保存を採用している。

## 6. よく使うオプション

### `default:` … デフォルト値を設定

新規レコード作成時に自動でセットされる値を指定できる。

```ruby
enum :status, { active: "active", suspended: "suspended" }, default: "active"
# User.new した時点で status が "active" になる
```

💡 補足: デフォルト値はモデルの enum 側でも、マイグレーションの `default:` でも設定できる。挙動の細かな差はあるが、初学者はまず「デフォルトを決められる」と覚えればOK。

### `prefix:` / `suffix:` … メソッド名に接頭辞・接尾辞をつける

複数の enum を持つモデルでメソッド名が衝突するのを防ぐ。

```ruby
enum :status, { active: "active", suspended: "suspended" }, prefix: true
# → user.status_active? / user.status_suspended? になる

enum :status, { active: "active", suspended: "suspended" }, suffix: :status
# → user.active_status? / user.suspended_status? になる
```

たとえば `status` と `role` の両方に `active` というキーがあると、`active?` メソッドがぶつかる。そういうときに prefix を使う。

### `validate:` … 不正な値をバリデーションエラーにする

```ruby
enum :status, { active: "active" }, validate: true
# 定義にない値をセットすると save 時にバリデーションエラー（例外ではなく errors に入る）
```

## 7. このプロジェクトの実例

### 例1: User（シンプルな2状態）

`app/models/user.rb`

```ruby
# アカウント状態（DB には文字列で保存）
# active:    通常（デフォルト）
# suspended: 管理者が一時的に利用を停止した状態（ログイン不可）
enum :status, {
  active: "active",
  suspended: "suspended"
}
```

ポイント: 各値の意味をコメントで説明している。「suspended だとログインできない」というビジネスルールが一目で分かる。

### 例2: Event（状態遷移を伴う4状態）

`app/models/event.rb`

```ruby
# ステータス（DB には文字列で保存）
enum :status, {
  draft: "draft",         # 下書き
  scheduled: "scheduled", # 予約公開（未来日時に自動公開）
  published: "published", # 公開中
  cancelled: "cancelled"  # 中止
}
```

ポイント: イベントは `draft → scheduled → published` のように状態が遷移する。enum を使うことで `event.published?` や `Event.scheduled` といった形で状態を扱える。

### 例3: EventApplication（集計のために状態を細かく分ける）

`app/models/event_application.rb`

```ruby
# applied:   申込済み（キャンセル・出席確認前の通常状態）
# cancelled: 参加者自身によるキャンセル済み
# rejected:  主催者都合によるキャンセル済み（参加者キャンセルと区別）
# attended:  出席確認済み（主催者がチェックインを行った状態）
# no_show:   無断欠席（rejected と区別して集計するため独立ステータス）
enum :status, {
  applied: "applied",
  cancelled: "cancelled",
  rejected: "rejected",
  attended: "attended",
  no_show: "no_show"
}
```

ポイント: 「キャンセル」を1つにまとめず、`cancelled`（参加者都合）と `rejected`（主催者都合）に分けている。集計や分析の都合で状態を細かく分けるのは実務でよくある設計判断。コメントにその意図（なぜ分けたか）が残っているのが良い例。

## 8. scope と組み合わせる（実務パターン）

enum が自動生成するスコープ（`Event.published` など）は、自分で定義した scope の中でも使える。これが実務で非常に強力。

### 例: Event の visible スコープ

`app/models/event.rb`

```ruby
# 公開中、または公開時刻を過ぎた scheduled を「表示対象」とする
scope :visible, -> {
  published.or(scheduled.where("publish_at <= ?", Time.current))
}
```

ここで `published` と `scheduled` は enum が自動生成したスコープ。それらを組み合わせて「公開中 または 公開予定時刻を過ぎた予約投稿」という複雑な条件を、読みやすく表現している。

### 例: EventApplication の upcoming / history スコープ

`app/models/event_application.rb`

```ruby
# 参加予定: applied かつ開催が未来
scope :upcoming, -> {
  joins(:event)
    .where(status: :applied)               # enum のキーをそのまま使える
    .where("events.start_at >= ?", Time.current)
    .order("events.start_at ASC")
}

# 履歴: 終了済みステータス、または開催が過去
scope :history, -> {
  joins(:event)
    .where(
      "event_applications.status IN (?) OR events.start_at < ?",
      %w[cancelled rejected attended no_show],  # 文字列で直接指定
      Time.current
    )
    .order("events.start_at DESC")
}
```

注目ポイント:

- `upcoming` では `where(status: :applied)` のように enum のキー（シンボル）で絞れる。
- `history` では `%w[cancelled rejected attended no_show]` と文字列の配列で指定している。生 SQL の `IN (?)` を使う場合は、enum の自動メソッドではなく実際に保存される文字列で書く必要があるため（文字列保存だからこそ素直に書ける）。

## 9. GraphQL / フロントとの連携

このプロジェクトはバックエンドが GraphQL（Rails）、フロントが TypeScript（Next.js）。enum の値はそのまま API を通じてフロントに渡る。

### 値の決定責任はバックエンドが持つ

Event のステータスは、フロントから直接受け取らず、バックエンドが決定している。

`app/models/event.rb`

```ruby
# action（"draft" or "publish"）と publish_at から status を決定する
# status の判定責任はバックエンドが持つ（フロントからは status を直接受け取らない）
def self.resolve_status(action:, publish_at:)
  if action == ACTION_DRAFT
    "draft"
  elsif publish_at.present?
    "scheduled"   # 公開日時が指定されていれば予約公開
  else
    "published"   # 即時公開
  end
end
```

なぜこうするのか: フロントから status を直接受け取ると、改ざんによって「本来 scheduled になるべきものを published にされる」といった不正が起きえる。enum の値（＝状態）はアプリの根幹なので、サーバー側で決める設計にしている。

これは enum 特有の話ではなく「重要な状態はサーバーが管理する」という一般的な設計原則。enum を使うときに意識しておくと良い考え方。

## 10. よくある落とし穴

### 落とし穴①: シンボルと文字列の混在

enum のキーはシンボルでも文字列でも指定できるが、混乱のもと。

```ruby
user.status = :active      # OK
user.status = "active"     # OK（同じ結果）
user.active?               # OK
```

どちらでも動くが、チーム内で書き方を揃えるのが無難。

### 落とし穴②: 存在しない値を入れると例外

```ruby
user.status = "deleted"   # enum に "deleted" は無い
# => ArgumentError: 'deleted' is not a valid status
```

`validate: true` を付けていない場合、**代入の時点で例外（ArgumentError）**が飛ぶ。フォームの値をそのまま代入するときは注意。

### 落とし穴③: 整数保存での値ズレ（再掲）

セクション5で説明した通り、整数保存で値の順番を変えると既存データが壊れる。このプロジェクトでは文字列保存に統一することで、この問題を根本的に回避している。

### 落とし穴④: bang メソッドはバリデーションを通る

```ruby
user.suspended!   # save! 相当 → バリデーション失敗時は例外
```

`user.suspended!` は内部で `save!` を呼ぶため、他のバリデーションに引っかかると例外が出る。「ステータスだけ変えたいのに、関係ないバリデーションで落ちる」ことがあるので、その場合は `update_column(:status, "suspended")`（バリデーションをスキップ）などを検討する。

### 落とし穴⑤: enum 名はカラム名と一致させる

```ruby
enum :status, { ... }    # ✅ カラム名 status に対応
```

enum の第1引数は**カラム名そのもの**。対応するカラムが無いとエラーになる（単数形/複数形が問題なのではなく、カラム名と一致しているかが本質）。

## 11. 新しい enum を追加する手順

実務で「新しい状態カラムを追加したい」となったときの手順。このプロジェクトの開発ルール（CLAUDE.md / docs/backend.md）に沿って説明する。

### ケースA: 新しい enum カラムをゼロから追加する

#### STEP 1. マイグレーションでカラムを追加（string 型）

文字列保存なので必ず `t.string` にする（`t.integer` ではない）。

```ruby
# db/migrate/YYYYMMDDNNNNNN_add_plan_to_users.rb
class AddPlanToUsers < ActiveRecord::Migration[8.1]
  def change
    add_column :users, :plan, :string, null: false, default: "free"
  end
end
```

⚠️ このプロジェクトのルール:

- リリース前は migration ファイルを直接書き換えて `db:drop db:create db:migrate`
- リリース後は新しい migration ファイルを足して `db:migrate`
- ファイル名は `YYYYMMDDNNNNNN_snake_case.rb`（当日の日付 + 連番）

#### STEP 2. モデルに enum を定義

```ruby
class User < ApplicationRecord
  # プラン（DB には文字列で保存）
  # free:    無料プラン（デフォルト）
  # premium: 有料プラン
  enum :plan, {
    free: "free",
    premium: "premium"
  }, default: "free"
end
```

コメントで各値の意味を書くこと（このプロジェクトの慣習）。

#### STEP 3. マイグレーション実行

```bash
docker compose exec backend rails db:migrate
```

#### STEP 4. 動作確認（Rails コンソール）

```bash
docker compose exec backend rails console
```

```ruby
u = User.first
u.free?          # => true
u.premium!       # => 保存される
User.premium     # => premium ユーザー一覧
```

### ケースB: 既存の enum に値を1つ追加する

文字列保存なら、これがとても簡単。**マイグレーション不要で、enum のハッシュに1行足すだけ。**

```ruby
enum :status, {
  active: "active",
  suspended: "suspended",
  banned: "banned"          # ← 1行足すだけ。既存データは無傷
}
```

（整数保存だと、足す位置によってはデータがズレるため慎重な対応が必要。文字列保存の恩恵が効くポイント。）

## 12. チートシート

User の `status` enum を例にした早見表。

```ruby
enum :status, { active: "active", suspended: "suspended" }
```

| やりたいこと | コード |
| --- | --- |
| 状態を確認 | `user.active?` / `user.suspended?` |
| 状態を変更して保存 | `user.suspended!` |
| 値をセット（保存しない） | `user.status = :suspended` |
| 絞り込み | `User.active` / `User.suspended` |
| 全選択肢を取得 | `User.statuses` |
| 現在の値を取得 | `user.status # => "active"` |
| where で絞る | `User.where(status: :active)` |

### 設計判断の早見表（このプロジェクトの方針）

| 判断 | このプロジェクトの選択 | 理由 |
| --- | --- | --- |
| 整数 vs 文字列保存 | 文字列（ハッシュ記法） | 値の追加・並び替えで既存データが壊れない |
| カラム型 | `t.string` | 文字列保存に対応 |
| 各値の説明 | コメントで明記 | ビジネスルール・設計意図を残す |
| 重要な状態の決定 | バックエンドが決定 | フロントからの改ざんを防ぐ |

## まとめ

- enum =「決められた選択肢の中の1つ」を安全に扱う仕組み
- 1行書くだけで述語メソッド・bang メソッド・スコープが自動生成される
- このプロジェクトは **ハッシュ記法で文字列保存**（`{ active: "active" }`）に統一 → 値の追加・並び替えに強い
- 各値の意味はコメントで残す、重要な状態決定はバックエンドが持つ
- 新しい値を足すだけなら1行追加で完了（文字列保存の恩恵）

まずは User のシンプルな例から読み、慣れたら Event（状態遷移）・EventApplication（集計向けの細分化）を読むと、実務での enum 設計の感覚がつかめる。

## 関連記事

- [graphql-ruby の Enum 型定義の読み解き（EventApplicationStatusType）](../graphql/2026-05-28-graphql-ruby-enum-type-definition.md) — こちらは **GraphQL（API）層の Enum 型**の話。本記事の「モデル/DB 層の enum」と対になる。モデルの `enum :status` をマスタにして GraphQL Enum を自動展開する設計（Single Source of Truth）まで読むと、enum がフロントまで一貫してどう流れるか分かる。
