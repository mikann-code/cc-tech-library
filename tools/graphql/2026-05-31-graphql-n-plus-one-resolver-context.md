---
title: GraphQL の N+1 問題と回避策（リゾルバで一括集計 → context 共有）
date: 2026-05-31
tags: [graphql, ruby, rails, n-plus-one, performance, context, resolver]
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

## 追記：なぜ「リゾルバの返り値に含める」ではなく context なのか

### そもそも「リゾルバの返り値に含める」とは？

リゾルバが返しているのは `EventApplication`（ActiveRecord オブジェクト）の配列。
これに no_show 累計を「含める」となると、たとえばこういう発想になる:

```ruby
# 案: 各 EventApplication に no_show 数をくっつけて返す
event.event_applications.map do |app|
  { application: app, no_show_count: counts[app.user_id] }
end
```

でもここで問題が起きる。

### 問題1: 返り値の「形」が型と合わなくなる

リゾルバの型宣言はこうなっている（`resolver:5`）:

```ruby
type [Types::OrganizerEventApplicationType], null: false
```

「`OrganizerEventApplicationType` の配列を返す」という契約。
`OrganizerEventApplicationType` は `EventApplication` モデルに対応した型なので、**1要素 = 1個の `EventApplication` オブジェクト** であることが前提。

上のように Hash や独自構造に変えてしまうと、型と噛み合わなくなり、`object.user_id` や `object.status` といった各フィールドのデフォルト解決（モデルのメソッド呼び出し）が壊れる。結局あちこち手当てが必要になる。

### 問題2: GraphQL の設計思想に逆らう

GraphQL は意図的に役割を分けている:

| 層 | 責務 |
| --- | --- |
| リゾルバ | 「どのオブジェクトを返すか」（取得・認可・並び替え） |
| Type | 「各オブジェクトの各フィールドをどう見せるか」（表現・計算） |

`organizerNoShowCount` は「`EventApplication` という1件の見え方の一部」なので、Type の責務。
リゾルバに計算結果を混ぜると、「取得」と「表現」が1か所に混ざってコードの見通しが悪くなる。

### 今の実装は「いいとこ取り」

```ruby
# リゾルバ: 集計を一括で“準備”だけする（取得の責務の範囲内）
context[:no_show_counts] = no_show_counts_by_user(organizer: user.organizer)

# Type: その準備済みデータを各行で“取り出す”（表現の責務）
def organizer_no_show_count
  (context[:no_show_counts] || {}).fetch(object.user_id, 0)
end
```

- 重い集計（DB アクセス）はリゾルバで1回だけ → N+1 を回避
- 各行への割り当ては Type → 型の構造を壊さない

context を仲介にすることで、「リゾルバの返り値の形は `EventApplication` のまま」「計算は Type」を両立している。

### 「リゾルバ返り値に含める」が向くケースもある

完全にダメなわけではない。たとえば:

- 返すデータがそもそも集計専用の独自構造（例:「ユーザーごとの統計サマリ」みたいな型）なら、リゾルバで組み立てた Hash/Struct を専用 Type にマッピングして返すのは自然
- その場合は専用の型を定義し、`type SomeStatsType` のように宣言する

つまり「返すものが既存モデル（`EventApplication`）そのものか / 独自の集計構造か」で選び方が変わる。
今回は「`EventApplication` の一覧 + 各行に付随する数字」なので、モデルはそのまま返し、付随情報は Type で足すのがきれい。

### まとめ

- リゾルバの返り値に直接含めるのは **不可能ではない** が、`EventApplication` の配列という型の契約を崩しやすく、GraphQL の「取得 ↔ 表現」の分離にも反する
- 今の実装は「集計の重い部分はリゾルバ（1回）、各行への適用は Type」というベストプラクティス。context がその橋渡し
- 「独自の集計型を返す」設計なら、リゾルバで組み立てて返すのもアリ。**要はデータの性質次第**
- 「数字を計算するのはリゾルバ、それを各行に見せるのは Type」と覚えると、この分担が腑に落ちる
