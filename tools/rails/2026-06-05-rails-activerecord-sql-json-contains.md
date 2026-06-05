---
title: Rails / ActiveRecord における SQL の知識まとめ（where と JSON_CONTAINS）
date: 2026-06-05
tags: [rails, active-record, ruby, sql, json, json-contains, security]
---

## 概要

ActiveRecord は普段の `User.where(role: "organizer")` の裏で常に SQL を組み立てている。普段はハッシュ記法で済むが、JSON 配列の中身を検索したいときなど「等価比較で書けないケース」では生 SQL（`where("JSON_CONTAINS(roles, ?)", ...)`）が登場する。本記事はその仕組みと、なぜ生 SQL が必要になるかをまとめる。

## サンプルコード

### 1. ActiveRecord は常に裏で SQL を作っている

DB（MySQL）が理解できる言語は SQL だけ。Ruby は直接理解できない。普段書く `User.where(role: "organizer")` も、内部で SQL に自動翻訳されて実行されている。

```
Ruby のコード ──（ActiveRecord が翻訳）──▶ SQL ──▶ MySQL が実行
```

```ruby
User.where(role: "organizer")
```

```sql
-- ↓ 自動翻訳
SELECT * FROM users WHERE role = 'organizer'
```

→ 「SQL を使っていなかった」のではなく、ActiveRecord が肩代わりしてくれていただけ。

### 2. where の2つの書き方

#### ① ハッシュ記法（普段よく使う）

```ruby
User.where(email: "a@example.com")
User.where(status: "published")
```

- `カラム名: 値` で書ける。読みやすく安全。
- 「値が〇〇と等しいか（`=`）」を調べる時はこれで足りる。

#### ② 生 SQL 文字列記法（特殊なケース）

```ruby
users.where("JSON_CONTAINS(roles, ?)", role.to_json)
```

- SQL を直接書く。ハッシュ記法で表現できない時の手段。

### 3. 生 SQL が必要になる場面

ハッシュ記法（`=` の比較）では書けない、以下のようなケースで登場する。

| やりたいこと | 例 |
| --- | --- |
| 大小比較（`>`, `<`, `<=`） | `where("publish_at <= ?", Time.current)` |
| JSON / 配列の中身検索 | `where("JSON_CONTAINS(roles, ?)", ...)` |
| OR / 複雑な条件 | `where("a = ? OR b = ?", x, y)` |
| LIKE 部分一致 | `where("title LIKE ?", "%#{q}%")` |
| SQL 関数を使う | `where("DATE(created_at) = ?", date)` |

→ 今までのコードが①ばかりだったのは、普通のカラムの完全一致がほとんどだったから。

### 4. JSON_CONTAINS とは

MySQL 8.0 の組み込み関数。**「JSON の中に指定した値が含まれているか」** を判定する。

```sql
JSON_CONTAINS(調べる対象, 探す値)
-- 含む → 1（true） / 含まない → 0（false）
```

例（`roles` が `["organizer", "participant"]` の場合）:

```sql
JSON_CONTAINS('["organizer","participant"]', '"organizer"')   -- → 1 ✅
JSON_CONTAINS('["organizer","participant"]', '"admin"')       -- → 0 ❌
```

### 5. なぜ JSON_CONTAINS が必要だったか（核心）

`roles` カラムが単一の値ではなく JSON 配列だから。

```
普通のカラム:  email = "a@x.com"            → = で比較できる
JSON配列:      roles = ["organizer", ...]    → "中に含まれるか" を見たい
```

- 配列 `["organizer","participant"]` は、文字列 `"organizer"` と `=` では一致しない。
- 「配列の中に含まれるか」を調べるには `JSON_CONTAINS` が必須。
- `roles` を配列にした理由 = 1人が「主催者」かつ「参加者」を同時に持てるから。

| 保存データ | `= "organizer"` | `JSON_CONTAINS` |
| --- | --- | --- |
| `["organizer"]` | ❌ | ✅ |
| `["organizer","participant"]` | ❌ | ✅ |

### 6. `role.to_json` が必要な理由

`JSON_CONTAINS` の第2引数（探す値）も JSON 形式でなければならない。

```ruby
role            # => organizer     （ただの文字列）
role.to_json    # => "organizer"   （JSON 文字列＝クォート込み）
```

```sql
JSON_CONTAINS(roles, '"organizer"')   -- ✅ 正しい JSON
JSON_CONTAINS(roles, 'organizer')     -- ❌ JSON じゃないのでエラー
```

### 7. プレースホルダ `?` = SQL インジェクション対策

値を文字列に直接埋め込まず、`?` 経由で渡す。

```ruby
# ❌ 危険：role に悪意ある値が入ると SQL を破壊できる
users.where("JSON_CONTAINS(roles, '#{role}')")

# ✅ 安全：? に ActiveRecord が安全にエスケープして埋める
users.where("JSON_CONTAINS(roles, ?)", role.to_json)
```

### 8. 全体のつながり（今回の学びの幹）

```
roles を「配列」で持ちたい（1人が複数ロール可）
   ↓
配列を DB に保存するには JSON 型カラムを使う
   ↓
JSON 配列の中身を検索するには JSON_CONTAINS が必要
   ↓
JSON_CONTAINS は ActiveRecord のハッシュ記法で書けない
   ↓
だから生 SQL（where("JSON_CONTAINS(roles, ?)", ...)）になる
```

「なんでここだけ SQL なの？」の答えは、結局すべて **「roles が配列だから」** に集約される。

## ポイント

- ActiveRecord は普段の `where(role: "organizer")` も裏で SQL に自動翻訳して実行している。「SQL を使っていなかった」のではなく、肩代わりしてくれていただけ。
- `where` には2つの書き方がある。①ハッシュ記法（`=` の等価比較）が普段使い、②生 SQL 文字列記法はハッシュ記法で表現できないときの手段。
- 生 SQL が必要になる典型は: 大小比較（`<=`）、JSON/配列の中身検索、OR/複雑条件、`LIKE` 部分一致、SQL 関数（`DATE()` など）。
- `JSON_CONTAINS(対象, 探す値)` は MySQL 8.0 の組み込み関数で、JSON の中に指定した値が含まれるかを 1/0 で返す。
- `roles` カラムが JSON 配列なので「中に含まれるか」を調べる必要があり、ハッシュ記法（`=`）では一致しない。だから `JSON_CONTAINS` が必須になる。
- 第2引数の `role.to_json` は、`JSON_CONTAINS` の探す値も JSON 形式でなければならないため。クォート付きの `"organizer"` にする必要がある。
- プレースホルダ `?` を必ず使う。文字列直埋め込みは SQL インジェクションの原因。`?` に渡すと ActiveRecord が安全にエスケープしてくれる。
- 結論: 「なぜここだけ生 SQL なのか」は「`roles` が配列カラムだから」の一点に集約される。配列で持つ → JSON 型 → 中身検索 → `JSON_CONTAINS` → ハッシュ記法で書けない → 生 SQL、という連鎖。
