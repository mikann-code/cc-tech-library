---
title: ActiveRecord の joins と includes の違い（繋ぐだけ／取得する）
date: 2026-06-02
tags: [rails, ruby, active-record, joins, includes, n-plus-one]
---

## 概要

ActiveRecord の `joins` は「繋ぐだけ」で関連先のデータを取得しない。
関連先のデータを実際に使いたい（取得したい）ときは `includes` を使う。

- `joins` … 関連テーブルを繋ぐ（その列を `where` や並び替えに使うため）。取得はしない
- `includes` … 関連データを取得する（実際に使う・N+1 回避）

## サンプルコード

### joins は「繋ぐ」だけ

```ruby
EventApplication.joins(:event).where(events: { organizer_id: 3 })
```

`joins(:event)` がやること:

- `events` テーブルを SQL 上で連結（JOIN）する → `events` の列を WHERE の条件に使えるようにする
- でも、`events` のデータを Ruby のオブジェクトとして持ってくるわけではない

生まれる SQL:

```sql
-- joins が生む SQL：events を繋ぐが、SELECT するのは event_applications だけ
SELECT event_applications.*           -- ← 取得するのは申込だけ
FROM event_applications
INNER JOIN events ON events.id = event_applications.event_id  -- ← 繋ぐ（条件用）
WHERE events.organizer_id = 3         -- ← events の列で「絞る」
```

つまり `joins` の目的は「`events` の列で絞り込みたいから繋ぐ」であって、`events` 自体を取得するためではない。

### 「取得」したいなら includes / preload

「上の階層（event）のデータも実際に使いたい（取得したい）」なら `includes` を使う。リゾルバの一覧取得で実際に使われている例:

```ruby
# organizer_event_applications.rb:22-23
event.event_applications
     .includes(user: :participant)   # ← user と participant を「取得」する（N+1回避）
```

### joins と includes の使い分け

| メソッド | 目的 | データを取得する？ |
| --- | --- | --- |
| `joins` | 関連テーブルの列で絞り込む / 並べる | ❌ しない（繋ぐだけ） |
| `includes` | 関連データを実際に使うために取得する | ✅ する（先読み） |

使い分けの例:

```ruby
# 絞りたいだけ → joins（events のデータは使わない、条件にするだけ）
EventApplication.joins(:event).where(events: { organizer_id: 3 })

# event の情報も画面で使う → includes（取得する）
EventApplication.includes(:event)
# このあと application.event.title などを使っても N+1 にならない
```

### no_show_counts で joins を使う理由

```ruby
EventApplication
  .joins(:event)                                       # events を繋ぐ
  .where(events: { organizer_id: organizer.id }, ...)  # events.organizer_id で絞るため
  .group(:user_id).count
```

ここで欲しいのは「event のデータ」ではなく「数えた結果（Hash）」だけ。
event は「主催者で絞る条件」に使うだけなので、取得不要 → `joins` が適切。
もし `includes` を使うと、不要な event データまで読み込んで無駄になる。

## ポイント

- `joins` = 関連テーブルを繋ぐ（その列で絞る・並べるため）。**取得はしない**
- `includes` = 関連データを取得する（実際に使う・N+1 回避）
- 「上の階層を取得」ではなく「上の階層を繋いで、その列を条件や並び替えに使う」が `joins` の正しい理解
- 集計だけしたい（`no_show_counts` のような GROUP BY → Hash）ケースは `joins` が正解。`includes` だと不要なデータを読んでしまう
- 区別の覚え方: 「`joins` = 条件に使うために繋ぐ／`includes` = 使うために取ってくる」

## 関連知識（補足）

### includes は内部で preload / eager_load を切り替える

- `preload(:event)` … 必ず別クエリで関連を取得（JOIN しない、2 クエリになる）
- `eager_load(:event)` … `LEFT OUTER JOIN` で 1 クエリにまとめて取得
- `includes(:event)` … Rails が状況を見てどちらかを自動選択

### includes + where で関連テーブルの列を使うときは references

`includes` だけだと preload（別クエリ）が選ばれることがあり、その状態で `where(events: { ... })` を書くと「そのテーブルが SQL に存在しない」エラーになる。
関連テーブルの列で絞る場合は `references` を併用するか、最初から `eager_load` を使う:

```ruby
# includes + where で関連列を使うなら references が必要
EventApplication.includes(:event).references(:event).where(events: { organizer_id: 3 })

# または最初から eager_load
EventApplication.eager_load(:event).where(events: { organizer_id: 3 })
```
