---
title: クラスメソッドとインスタンスメソッド — scope と def の違い
date: 2026-06-26
tags: [ruby, rails, scope, class-method, instance-method]
---

## 概要

論理削除コードを題材に、Ruby のクラスメソッド（`scope`）とインスタンスメソッド（`def`）の違いをまとめたもの。同名でも共存できる理由と、衝突するケースを整理する。

## サンプルコード

題材となるモデル定義:

```ruby
class Wordbook < ApplicationRecord
  scope :kept,      -> { where(deleted_at: nil) }       # クラスメソッド
  scope :discarded, -> { where.not(deleted_at: nil) }   # クラスメソッド

  def discarded?                                         # インスタンスメソッド
    deleted_at.present?
  end
end
```

呼ぶ相手で結果が変わる:

```ruby
Wordbook.discarded   # クラス（全体）に呼ぶ → 削除済みの“一覧”
book = Wordbook.find(1)
book.discarded?      # 1件に呼ぶ → その1冊が削除済みか（true / false）
```

`?` を外して両方 `discarded` にしても衝突しない:

```ruby
# ? を外して両方 discarded にしても衝突しない
scope :discarded, -> { ... }   # Wordbook.discarded
def discarded; deleted_at.present?; end   # book.discarded
```

逆に「衝突する」のはどんなとき:

```ruby
# クラス同士で同名 → 衝突（後勝ちで上書き）
scope :discarded, -> { ... }
def self.discarded; ...; end

# scope の二重定義 → 衝突
scope :discarded, -> { ... }
scope :discarded, -> { ... }
```

## ポイント

- 一番大事な軸は「誰に対して呼ぶか」。`Wordbook.〜`（全体に聞く）= 一覧、`book.〜`（1冊に聞く）= 単体。
- クラスメソッド = `Wordbook`（モデル＝全体）に呼ぶ。`scope` は実質 `def self.xxx`。返すのはレコードの一覧（Relation）。テーブル全体に問い合わせるイメージ。
- インスタンスメソッド = `book`（1件のオブジェクト）に呼ぶ。`def xxx`。返すのはその1件の値・真偽値。手元の1冊に問いかけるイメージ。
- 同じファイルに同居できるのは、`scope`（クラス側）と `def`（インスタンス側）で Ruby の置き場所そのものが別だから。階層が違うので名前が被っても上書きし合わない。
- `discarded?` の `?` は「真偽値（true / false）を返すメソッドには `?` を付ける」という Ruby の命名慣習。`present?` や `nil?` と同じ。衝突回避のためではない。
- 衝突するかどうかは `?` の有無ではなく「同じ階層で同名か」で決まる。クラスメソッド（scope）× インスタンスメソッド（def）は衝突しない。クラスメソッド × クラスメソッド、インスタンスメソッド × インスタンスメソッドは衝突する。
