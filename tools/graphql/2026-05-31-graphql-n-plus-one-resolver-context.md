---
title: GraphQL の N+1 問題と回避策（リゾルバで一括集計 → context 共有）
date: 2026-05-31
tags: [graphql, ruby, rails, n-plus-one, performance, context]
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

## 追記：context の役割を中心に整理

### context とは

1回のリクエストの間だけ生きる、共有の箱（Ruby の Hash）。
リゾルバ・型・Mutation など離れた場所どうしでデータを共有するために使う。

### どこで作られるか

`graphql_controller.rb:13-17` で初期メンバーを定義し、`AppSchema.execute(..., context: context)` でエンジンに渡す。

```ruby
context = {
  current_user: current_user,   # ログイン中のユーザー
  session: session,             # セッション
  controller: self,             # reset_session 用
}
```

### 性質（重要な3つ）

| 観点 | 性質 |
| --- | --- |
| 1リクエスト内 | context は 1個。全員（リゾルバ・型・Mutation）で共有 |
| 複数リクエスト | リクエストごとに 別の1個（同時アクセスでも混ざらない） |
| 中の要素（キー） | 何個でも追加できる |
| 寿命 | リクエストが終わると消える（次には引き継がれない） |

### 使い方：読むだけでなく「書き足せる」

```ruby
# リゾルバで要素を追加（organizer_event_applications.rb:20）
context[:no_show_counts] = no_show_counts_by_user(organizer: user.organizer)

# 型で取り出す（organizer_event_application_type.rb:25）
(context[:no_show_counts] || {}).fetch(object.user_id, 0)
```

### なぜ context を使うのか

リゾルバから型へは引数で渡せないから。
両者はお互いを直接呼ばず、GraphQL エンジンが間に立つ。共有できる唯一の手段が context。

```
リゾルバ → context[:no_show_counts] に置く ─┐
                                          │ 同じ箱を共有
型     → context[:no_show_counts] を読む ─┘
```

### 型が「人数ぶん」呼ばれる理由

| 主体 | 呼ばれる回数 | 理由 |
| --- | --- | --- |
| リゾルバ | 1回 | 「一覧を作る」処理。一覧は1個 |
| 型 | N回（人数ぶん） | 「1人ぶんの表示」を作る処理 |

→ 型の中でDBを叩くと、人数ぶんSQLが飛ぶ。

### リゾルバと型の役割分担

| 主体 | この値を | 役割 |
| --- | --- | --- |
| リゾルバ | 作って context に置く（自分は使わない） | 生産者 |
| 型 | context から取り出して使う | 消費者 |

### 速くなる理由（DB往復の観点）

SQL 1回には アプリ ⇄ DB の往復時間がかかる。回数が減るほど往復が減り、速くなる。

```
処理時間
  │          ╱ N+1（人数に比例して遅い）
  │       ╱
  │ ━━━━━━━━ 一括（人数が増えても一定で速い）
  └────────── 申込者数
```

N+1 は開発中は気づきにくく、本番でデータが増えると急に遅くなるのが怖い点。

### 本質

DBへの往復回数をいかに減らすか。
リゾルバ（1回呼ばれる）で一括集計し、context で型（人数ぶん呼ばれる）に渡せば、SQLが「人数分 → 2回固定」になり、DB往復が減って格段に高速化する。
