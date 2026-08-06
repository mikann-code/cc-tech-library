---
title: development だけ :memory_store の env 差分は、レート制限を入れた瞬間に負債になる
date: 2026-07-31
tags: [rails, solid-cache, rack-attack, dev-prod-parity, cache-store]
source: 実務(Rails)
---

## 概要

キャッシュストアの選択は「保存先」ではなく「共有スコープ」の選択。Rails 8 のデフォルトは development が `:memory_store`、production が `:solid_cache_store` と env ごとにバラバラで、`Rails.cache` を使う機能が無いうちは無害に見える。しかし rack-attack のようにカウンタを `Rails.cache`（`Rack::Attack.cache.store` のデフォルト）に置く機能を 1 つ入れた瞬間、dev/prod のストア差分がレート制限の正しさそのものに直結する。

## サンプルコード

単一 DB 構成で development も `:solid_cache_store` に揃える場合、通常の migration でプライマリ DB に `solid_cache_entries` を作る。カラム型・インデックスは gem の `db/cache_schema.rb` と一致させる（ズレると「dev では通るが本番で落ちる」を自作することになる）。

```ruby
# 単一DB構成で solid_cache_entries を作る最小例
class CreateSolidCacheEntries < ActiveRecord::Migration[8.0]
  def change
    create_table :solid_cache_entries do |t|
      t.binary  :key,        null: false, limit: 1024
      t.binary  :value,      null: false, limit: 536_870_912
      t.datetime :created_at, null: false
      t.integer :key_hash,   null: false, limit: 8
      t.integer :byte_size,  null: false, limit: 4

      t.index :key_hash, unique: true
      t.index :byte_size
      t.index [:key_hash, :byte_size]
    end
  end
end
```

test 環境のストアはデフォルトで `:null_store`（何も保存しない）なので、`Rails.cache` 越しには移行を検証できない。ストアを明示的に組み立てて叩く。

```ruby
store = ActiveSupport::Cache.lookup_store(:solid_cache_store)

store.write("counter", 0, raw: true)
expect(store.increment("counter")).to eq(1)
expect(store.increment("counter")).to eq(2)
```

## ポイント

- **キャッシュストアの選択は「保存先」ではなく「共有スコープ」の選択**。「高速化」用途は共有されなくてもキャッシュミスが増えるだけで壊れないが、「カウンタ・一時状態の保持」用途（レート制限など）ではストア選択が仕様の一部になる。
- `:memory_store` は**プロセスローカル**。本番のマルチプロセス／多台構成では各プロセスが別々に数え、「5 回/分」が実質「5 回 × プロセス数」になりレート制限が成立しない。development は単一プロセスなので「動いて見えて」壊れに気づけない。
- 対策は development も `:solid_cache_store`（DB ベース＝プロセス間で共有）に揃える。`config/cache.yml` の該当環境に `database:` 指定が無ければ Solid Cache はプライマリ接続を使うので、単一 DB でも設定を足さず動く。スキーマは gem 定義とカラム型・インデックスを一致させる（例: MySQL では `limit: 536_870_912` が `longblob`、`limit: 8` の integer が `bigint`）。
- 検証は `ActiveSupport::Cache.lookup_store(:solid_cache_store)` で development と同じストア実装を test から直接叩く。`connection.columns` / `connection.indexes` で gem のスキーマ定義との一致を固定するテストは、単なる存在チェックより回帰防止になる。
- **「今は誰も使っていない設定」の env 差分は時限爆弾**。共有スコープ・永続性・シリアライズ可否・容量上限の 4 点は dev では差が見えず本番だけで顕在化する。新機能がその設定に依存し始めたときが、env 差分を解消する自然なタイミング。
