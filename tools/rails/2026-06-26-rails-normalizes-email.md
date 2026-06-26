---
title: Rails の normalizes による email 正規化（大小無視のユニーク制約と3層整合）
date: 2026-06-26
tags: [rails, normalizes, email, uniqueness, active-record]
---

## 概要

Rails 7.1+ の `normalizes` 機能で、`email` に値が代入されたとき・保存される前に、自動で値を整形する設定。前後空白の除去（strip）と小文字化（downcase）を行う。

## サンプルコード

```ruby
# 保存前に email を正規化（前後空白除去 + 小文字化）。
# DB の照合順序が ci（大小無視）なので、アプリ側も大小無視で揃える。
normalizes :email, with: ->(email) { email.strip.downcase }
```

## 何をしているか

Rails 7.1+ の `normalizes` 機能で、`email` に値が代入されたとき・保存される前に、自動で値を整形する設定。

| 処理 | 例 |
| --- | --- |
| `strip` … 前後の空白を除去 | `" foo@x.com "` → `"foo@x.com"` |
| `downcase` … 小文字化 | `"Foo@X.com"` → `"foo@x.com"` |

代入時・保存前だけでなく、`User.find_by(email: "Foo@X.com")` のような検索時にも正規化が効くので、入力ブレを一貫して吸収できる。

## なぜ必要か（3層で整合を取る設計）

| 層 | 実装場所 | 役割 |
| --- | --- | --- |
| アプリ（正規化） | `user.rb:15` `normalizes` | 保存値を小文字に統一 |
| アプリ（検証） | `user.rb:17` `uniqueness: { case_sensitive: false }` | 大小無視で重複を弾く |
| DB | `schema.rb:70` unique index + 照合順序 `utf8mb4_0900_ai_ci` | DB レベルで大小無視のユニーク制約 |

ポイントは DB の unique index が大小無視の照合順序の上に乗っていること。Rails の uniqueness 検証はアプリレベルなので単体だと競合（race condition）で重複が入りうるが、DB の unique index が最後の砦になり、`Foo@x.com` と `foo@x.com` の同時登録も弾ける。コメントの「DB が ci だからアプリ側も揃える」は実際に成立している。

> ⚠️ 補足：照合順序 `utf8mb4_0900_ai_ci` の `ai` は accent-insensitive（アクセント無視）も意味する。email では実害ないが、コメントの「大小無視」より少し広い挙動（é=e）になっている点だけ留意。

## メールアドレスの大小文字の関係

メアドは ローカル部@ドメイン部 で、大小文字の扱いが両者で異なる。

### ドメイン部（@example.com の部分）

- DNS 仕様で大小無視。`Example.com` と `example.com` は同一サーバーを指す。
- これは設計判断ではなくインターネットの事実。「大小だけ違う別ドメイン」は実在しない。
- → downcase しても情報は一切失われない（完全に安全）。

### ローカル部（user@ の部分）

- RFC 5321 上は大小区別が許されている（理論上 `User@` と `user@` を別メールボックスにできる）。
- ただし現実には Gmail / Outlook などほぼ全プロバイダが大小無視で運用。大小だけ違う2つを別人扱いする事業者は実質存在しない。
- → 大小無視で同一とみなすのは業界標準の安全な割り切り。

### まとめ表

| 部分 | 仕様上 | 実運用 | 大小無視は安全か |
| --- | --- | --- | --- |
| ドメイン部 | 大小無視（DNS） | 大小無視 | ✅ 完全に安全（事実） |
| ローカル部 | 大小区別が許される | ほぼ大小無視 | ✅ 実用上安全（標準的割り切り） |

## このアプリでの帰結

downcase でローカル部・ドメイン部をまとめて小文字化して保存するため、

- `User@Example.com` で入力 → DB には `user@example.com` で保存
- 後から `user@example.com` で登録 → 同値なので unique 制約で弾かれる

→ このシステム内では「大小だけ違うメアド」は別アカウントとして共存できない。email の用途としてはこれが正しい挙動。
