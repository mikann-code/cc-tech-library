---
title: GraphQL の N+1 問題と回避策（リゾルバで一括集計 → context 共有）
date: 2026-05-31
tags: [graphql, ruby, rails, n-plus-one, performance]
---

## 概要

主催者のイベント申込者一覧で、各申込者に「要注意者バッジ」を表示する。
バッジの中身 = その申込者が「ログイン中の主催者のイベント」で 無断欠席 (no_show) にされた累計回数。

対象フィールド: `organizer_event_application_type.rb:15 organizer_no_show_count`

## N+1 問題とは

申込者一覧（N人）を返すと、GraphQL は 1人ずつ 型のフィールドを解決する。
ここで「1人ずつ no_show を COUNT」すると SQL が人数分発行される。

```
① 申込者一覧の取得            → 1 クエリ
② 各申込者の no_show を COUNT  → N クエリ（人数ぶん）
────────────────────────────────────
合計 1 + N クエリ
```

例: 申込者100人 → 101回 SQL が飛ぶ。人数に比例して遅くなる。

## サンプルコード

### ❌ アンチパターン（型側で毎回COUNT）

```ruby
def organizer_no_show_count
  EventApplication
    .where(user_id: object.user_id, status: :no_show)
    .count   # ← 申込者ごとに SQL 発行 = N+1 の原因
end
```

### ✅ 回避策 — リゾルバで一括集計 → context で渡す

申込者を解決する前に、`GROUP BY user_id` で 1クエリにまとめて集計し、context に載せておく。

事前集計: `organizer_event_applications.rb:20`

```ruby
# user_id => no_show回数 のハッシュを1クエリで作成
EventApplication
  .joins(:event)
  .where(events: { organizer_id: organizer.id }, status: :no_show)
  .group(:user_id)
  .count
# => { 5 => 2, 12 => 1 }   # user_id 5番は2回、12番は1回
```

型側は SQL を投げず、メモリ上のハッシュを引くだけ:

```ruby
def organizer_no_show_count
  return 0 unless context[:current_user]&.organizer?
  (context[:no_show_counts] || {}).fetch(object.user_id, 0)
end
```

### クエリ数の比較

```
① 申込者一覧の取得   → 1 クエリ
② no_show 一括集計    → 1 クエリ
────────────────────────────
合計 2 クエリ（固定）
```

申込者が何人いても 2回で一定。N+1 が解消される。

## ポイント

| 観点 | 内容 |
| --- | --- |
| 問題 | 申込者ごとに COUNT すると 1 + N クエリ |
| 原因 | GraphQL が申込者を1人ずつ解決するため |
| 対策 | リゾルバで GROUP BY 一括集計し `context[:no_show_counts]` に事前ロード |
| 効果 | クエリ数が 2回固定（人数に依存しない） |
| 注意 | no_show 0回の人はハッシュに無い → `fetch(user_id, 0)` で 0 補完 |

### 0 フォールバックの理由

`GROUP BY + COUNT` の結果には「no_show が1度もない人は行として現れない」ため、ハッシュにキーが存在しない。
そのため型側で `fetch(object.user_id, 0)` の第2引数 `0` を使い、「キーが無ければ 0 回」とフォールバックしている。
