---
title: Ruby のキーワード引数とローカル変数まとめ
date: 2026-05-15
tags: [ruby, keyword-arguments, local-variables, method-arguments, instance-variables, rails]
---

## 概要

Ruby のキーワード引数とローカル変数の関係についてのまとめ。`:` の有無の意味、引数宣言時とローカル変数として使うときの違い、`update!(email: email)` の左右の役割の違い、Ruby 3.1+ の省略記法、デフォルト値による必須/任意の挙動まで。

## 1. キーワード引数の基本

引数名の後ろにコロン（`:`）を付けると キーワード引数 になる。

```ruby
def resolve(current_password:, email:)
  # ...
end

resolve(current_password: "password1", email: "new@example.com")
```

| 場所 | 役割 |
| --- | --- |
| 左の `email:` | 引数の名前 |
| 右の `"new@example.com"` | 渡す値 |
| 全体（`email: "..."`） | これで1つのキーワード引数 |

## 2. 位置引数とキーワード引数の違い

| 観点 | 位置引数 | キーワード引数 |
| --- | --- | --- |
| 定義 | `def foo(a, b)` | `def foo(a:, b:)` |
| 呼び出し | `foo("x", "y")` | `foo(a: "x", b: "y")` |
| 順番 | 重要（順番通り） | 自由（名前で識別） |
| 可読性 | 引数が多いと分かりにくい | 何の値か明確 |

> プロジェクトのルール: 引数が2つ以上ならキーワード引数を使う（CLAUDE.md）

## 3. `:` を使う場面・使わない場面

`:` は 「キーワード引数ですよ」というマーク。

```ruby
def resolve(email:)              # ① 引数宣言         → : あり
  puts email                     # ② ローカル変数として使用 → : なし
  user.update!(email: email)     # ③ 別メソッドに渡す      → 左 : あり / 右 : なし
end
```

| 場面 | 書き方 | `:` |
| --- | --- | --- |
| メソッド定義で引数宣言 | `def foo(email:)` | あり |
| メソッド内でローカル変数として使う | `puts email` | なし |
| 別メソッドにキーワード引数として渡す | `bar(email: email)` | 左のみあり |

## 4. メソッドの引数 = ローカル変数になる仕組み

引数を宣言すると、呼び出し時に渡された値が自動でローカル変数に代入される。

```ruby
def resolve(current_password:, email:)
  # この時点で current_password と email が
  # ローカル変数として使える状態になっている
  puts email
end

resolve(current_password: "xxx", email: "yyy")
# ↓ Ruby が裏でやっていること（イメージ）
# current_password = "xxx"
# email = "yyy"
```

ポイント: 引数の宣言 = 「外部から値を受け取る入り口つきのローカル変数を作る」

## 5. ローカル変数のスコープ

「ローカル」= そのメソッドの中だけで有効。

```ruby
def resolve(email:)
  puts email   # ✅ OK：このメソッド内なので使える
end

def other_method
  puts email   # ❌ エラー：別メソッドからは見えない
end
```

引数で受け取ったものも、メソッド内で `=` で作ったものも、どちらも 同じ「ローカル変数」。

```ruby
def resolve(email:)
  # ① 引数として受け取ったローカル変数
  puts email

  # ② メソッド内で自作したローカル変数
  message = "Hello"
  puts message
end
```

## 6. `update!(email: email)` の左右の違い

同じ単語が2回出てくるが、役割は別物。

```ruby
user.update!(email: email)
#            ━━━━━  ━━━━━
#            左      右
#            User の  resolve の引数で受け取った
#            カラム名 ローカル変数の値
```

|  | 左 `email:` | 右 `email` |
| --- | --- | --- |
| 何を指すか | DB の カラム名 | ローカル変数 |
| 変えられるか | 変えられない（DB 構造に従う） | 変えられる（引数名による） |
| `:` | あり（キーワード引数として渡すので） | なし（ただの変数参照なので） |

## 7. Ruby 3.1+ の省略記法

左右が同名なら値を省略できる。

```ruby
# 通常
validate(user: user, current_password: current_password, email: email)

# Ruby 3.1+ の省略記法
validate(user:, current_password:, email:)
```

実例: `update_user_email.rb:15`

## 8. デフォルト値の有無で必須/任意が決まる

「キーワード引数 ≠ 必須」。デフォルト値の有無で決まる。

| 書き方 | 種類 | 必須/省略可能 |
| --- | --- | --- |
| `def foo(a)` | 位置引数 | 必須 |
| `def foo(a = "x")` | 位置引数 | 省略可能 |
| `def foo(a:)` | キーワード引数 | 必須 |
| `def foo(a: "x")` | キーワード引数 | 省略可能 |

## 一言で覚えるなら

`:` は「キーワード引数として扱う」マーク。引数の宣言は「外部から値を受け取るローカル変数を作る」こと。

## 追記：インスタンス変数（`@`）と View への受け渡し

> §5「ローカル変数のスコープ」の続編。`@` を付けるとローカル変数ではなくインスタンス変数になり、Mailer / Controller から View にも値が渡る。

### 概要

```ruby
@user = user
@event = event
@body_text = body_text
```

| 観点 | 内容 |
|---|---|
| `@` を付ける | インスタンス変数になる |
| インスタンス変数の特徴 | 同じインスタンス内のどこからでも参照できる |
| Mailer / Controller での使われ方 | View（テンプレート）からも参照可能になる |

つまり「`@` を付けると View に渡せる」という直感はそのとおり。

### 仕組みを少し詳しく

#### `@` なし（ローカル変数）

```ruby
def notify(user:, ...)
  user.email   # ← このメソッド内でだけ使える
end
```

→ メソッドの外（View など）からは見えない。

#### `@` あり（インスタンス変数）

```ruby
def notify(user:, ...)
  @user = user   # ← インスタンス全体で共有
end
```

→ View からも `@user` として参照可能。

### View で実際に使う

`app/views/event_message_mailer/notify.html.erb`（こんな感じのファイルがあるはず）:

```erb
<%= @user.name %>様

<%= @body_text %>

【イベント情報】
<%= @event.title %>
日時: <%= @event.start_at %>
```

| View 内 | 参照元 |
|---|---|
| `@user.name` | Mailer で `@user = user` した値 |
| `@body_text` | Mailer で `@body_text = body_text` した値 |
| `@event.title` | Mailer で `@event = event` した値 |

→ `@` で渡したものだけが View で使える。

### なぜ `subject_text` は `@` にしないのか（再確認）

```ruby
mail(to: user.email, subject: "【#{SERVICE_NAME}】#{subject_text}")
```

| 変数 | 使われ方 | `@` 必要? |
|---|---|---|
| `subject_text` | メソッド内で件名文字列を組み立てるだけ。View では使わない | ❌ 不要 |
| `body_text` | View で本文として表示 | ✅ 必要 |

→ **View で参照する変数だけ `@` にする**、というのが Rails の慣習。

### Controller でも同じ

実はこれは Mailer 限定の話ではなく、Controller でも全く同じ仕組み:

```ruby
class EventsController < ApplicationController
  def show
    @event = Event.find(params[:id])   # ← @ にセット
  end
end
```

```erb
<%# events/show.html.erb %>
<h1><%= @event.title %></h1>          <%# View で参照 %>
```

→ Rails の **「Controller / Mailer のインスタンス変数は自動的に View に渡る」** という規約。

### まとめ

| 質問 | 答え |
|---|---|
| `@` を付けるとインスタンス変数になる? | ✅ 正しい |
| View に送れる? | ✅ 正しい（Mailer / Controller の場合） |
| `@` なしだと? | メソッド内のローカル変数。View からは見えない |
| いつ `@` を付ける? | View で参照したい値だけ |

### 一行で言うと

> `@` = Mailer / Controller から View へ値を渡すためのスイッチ

### 追記分のポイント

- `@` を付けるとローカル変数ではなく **インスタンス変数** になり、同じインスタンス内のどこからでも参照できる
- Mailer / Controller では、インスタンス変数が **自動的に View に渡される**（Rails の規約）
- **View で参照したい変数だけ `@` を付ける** のが慣習（件名のように内部で組み立てるだけのものは `@` 不要）
- 同じ仕組みは Controller でも Mailer でも共通
