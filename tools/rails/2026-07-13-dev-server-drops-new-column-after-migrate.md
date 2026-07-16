---
title: 起動中の Rails dev サーバーは migrate 後の新カラムを「例外なしで」黙って保存しない
date: 2026-07-13
tags: [rails, activerecord, migration, docker, schema-cache]
source: 実務(Rails)
---

## 概要

Rails の dev サーバーを起動したまま別プロセスで migrate してカラムを追加すると、起動中プロセス経由の書き込みだけがおかしくなる。`update!` しても例外にならず in-memory では代入した値が見えるのに、発行される UPDATE 文から新カラムが黙って落とされ、DB は NULL のまま。migrate 後はサーバーと job worker のプロセス再起動が必須になる。

## サンプルコード

```bash
# dev サーバーを起動したまま migrate すると起きる
docker compose exec backend rails db:migrate
```

```ruby
# 追加したマイグレーション（最小例）
add_column :items, :processed_at, :datetime
```

```ruby
# 一見正常に動くが、processed_at が UPDATE 文に含まれない
item.update!(status: :done, processed_at: Time.current)
```

```sql
-- 実際に発行された UPDATE（processed_at が入っていない・例外も出ない）
UPDATE `items` SET `status` = 'done', `updated_at` = '...' WHERE `items`.`id` = 1
```

```bash
# 解決: migrate 後に長生きプロセスを再起動する（worker も忘れず）
docker compose restart backend worker
```

## ポイント

- 例外は出ず（`ActiveModel::UnknownAttributeError` にならない）、直後の `item.processed_at` は in-memory の値を返すため、API レスポンスだけ見ていると気づけない。同じ `update!` の他カラム（`status`）は正しく保存されるのも罠。
- 新しいプロセス（`rails runner` や RSpec）では同じコードが正常に保存される。＝ DB / マイグレーション自体は正しく、**起動中プロセスが見ているスキーマ情報の鮮度**が問題。
- 推定原因は、起動中プロセスの ActiveRecord スキーマキャッシュ（接続 / コネクションプール側のカラム情報）が migrate 前の古い状態のまま残ること。確実に言えるのは「起動中プロセス経由の書き込みだけが新カラムを落とす」「新プロセスは正常」「再起動で解消」の3点。
- 対策は **migrate したら長生きプロセスを全部再起動する**を習慣化すること。dev サーバーだけでなく job worker も対象（worker も古いスキーマ情報を持ち続けうる）。
- 「新プロセスでは通るのに起動中サーバー経由だけ落ちる」E2E テストが出たら、コードではなくプロセスのスキーマ鮮度を疑う。切り分けには DB を直接見る / 発行された SQL を見るのが決め手になる。
