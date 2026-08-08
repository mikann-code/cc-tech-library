---
title: ActiveRecord Encryption を deterministic で全面採用すると、SQL でできるのは「等価一致」だけになる
date: 2026-08-06
tags: [rails, activerecord, encryption, security, sql]
source: 実務(Rails)
---

## 概要

Rails 7 以降で標準の ActiveRecord Encryption（`encrypts`）で個人情報カラムを暗号化する際、検索条件に使うカラムが多かったため `deterministic: true` を全面採用した。決定的暗号化は「同じ平文 → 同じ暗号文」なので等価一致（`where(status: 'RUNNING')`）と DB インデックスは効く。しかしその裏返しとして、範囲比較・並び替え・部分一致・`update_all` が軒並み使えなくなり、「暗号化するカラムを決める」ことが「そのカラムでどんなクエリを書くかを決める」ことと同義だと分かった。

## サンプルコード

暗号文は文字列としてカラムに入るため、復号後の型は `attribute` を **`encrypts` より先に** 宣言する（マイグレーションではカラムを `text` にする）。

```ruby
class Item < ApplicationRecord
  attribute :started_on, :date
  attribute :score, :integer

  encrypts :started_on, :score, deterministic: true
end
```

`update_all` は暗号化を通らないので、一括更新は 1 件ずつに落とす。

```ruby
Item.where(status: 'PENDING').find_each { |i| i.update!(status: 'DONE') }
```

## ポイント

- **`deterministic: true` を選ぶのは、そのカラムのクエリ仕様を「等価一致のみ」に確定させる意思決定**。カラム単位で「暗号化するか」ではなく「そのカラムでどんな WHERE を書くか」を先に洗い出して決める（等価一致だけで足りる → deterministic ／ 検索に使わない → 非決定的の方が安全 ／ 範囲・並び替え・部分一致が必要 → 暗号化しない設計か別手段）。
- 決定的暗号化が保証するのは「平文の等価性 → 暗号文の等価性」だけ。順序・大小・部分一致は保存されないので `<` `>` `BETWEEN` `ORDER BY` `LIKE` は暗号文の辞書順比較になり無意味な結果を返す。日付・数値カラムを暗号化した時点で SQL の期間・閾値判定はできなくなる。
- `update_all` / `insert_all` は SQL に直行し Attributes API（型キャスト＋暗号化）を通らないため壊れる。一括更新は `find_each` + `update!` に置き換え、1 件ずつ UPDATE のコストは受け入れる。
- 範囲比較がどうしても要るなら「1 レコードあたり数件」と保証できる小さいスコープに限って、復号後に Ruby 側のメソッドで判定する（一覧全体に対してやるとメモリを食い N+1 も踏む）。暗号化カラムで絞り込む一覧は設計段階で潰しておくのが正解。
- 鍵（`primary_key` / `deterministic_key` / `key_derivation_salt`）は最初に固定する。決定的暗号文は鍵と salt から決まるため、後から変えると既存データを等価一致で引けなくなる（実質データ移行）。移行互換の `support_unencrypted_data` は移行完了後に false へ戻す。

## 参考

- [Active Record Encryption — Ruby on Rails Guides](https://guides.rubyonrails.org/active_record_encryption.html)
