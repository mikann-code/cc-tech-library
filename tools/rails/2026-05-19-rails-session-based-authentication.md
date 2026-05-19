---
title: Rails のセッションベース認証の仕組み（DB セッションストア）
date: 2026-05-19
tags: [rails, authentication, session, cookie, activerecord-store]
---

## 概要

このプロジェクトは **DB ベースのセッション** で認証を管理している。JWT でも cookie store でもなく、**サーバー側の MySQL にセッションデータを保存**し、ブラウザには `session_id` だけを cookie で渡す設計。

> Redis を使わずに済むため、インフラがシンプル（設計方針: 「Redis 不使用 — ジョブキュー・セッション・キャッシュはすべて MySQL で管理」）

## サンプルコード

### シーケンス

```
[ブラウザ]                  [Rails Middleware]              [Rails App]                  [MySQL]
    │                             │                              │                          │
    │ ① POST /graphql             │                              │                          │
    │  Cookie: _app_session=abc   │                              │                          │
    ├────────────────────────────►│                              │                          │
    │                             │ ② session_id で sessions テーブルを引く                  │
    │                             ├────────────────────────────────────────────────────────►│
    │                             │                              │  ◄── { user_id: 42 } ────┤
    │                             │ ③ session ハッシュを復元                                  │
    │                             ├─────────────────────────────►│                          │
    │                             │                              │ ④ current_user で          │
    │                             │                              │   User.find_by(id: 42)   │
    │                             │                              ├─────────────────────────►│
    │                             │                              │  ◄────── User ───────────┤
    │                             │                              │ ⑤ context に詰める          │
    │                             │                              │ ⑥ resolver が context から │
    │                             │                              │   current_user を返す     │
    │  ◄──────────────────────────┴──────────────────────────────┤                          │
```

### 1. セッション機能を有効化

`config/application.rb`

```ruby
config.middleware.use ActionDispatch::Cookies
config.middleware.use ActionDispatch::Session::ActiveRecordStore,
  key: "_app_session",       # cookie の名前
  same_site: :lax,
  secure: Rails.env.production?,
  domain: ENV["SESSION_COOKIE_DOMAIN"]
```

Rails は **API mode** だとデフォルトでセッション機能を持たないため、middleware を **手動で追加**して `session` ハッシュを使えるようにしている。

### 2. ログイン時にセッションへ user_id を保存

`app/graphql/mutations/login.rb`

```ruby
context[:session][:user_id] = user.id
```

この 1 行で起きること:

| # | 動作 | 場所 |
|---|------|------|
| 1 | session ハッシュに `{ user_id: 42 }` を保存 | メモリ |
| 2 | レスポンス時に `ActiveRecordStore` middleware が走る | - |
| 3 | `sessions` テーブルに INSERT（session_id + 暗号化された data） | MySQL |
| 4 | `Set-Cookie: _app_session=abc123` をレスポンスに付与 | HTTP |
| 5 | ブラウザが cookie として保存 | ブラウザ |

> **重要**: cookie には `session_id`（ランダム文字列）のみ。`user_id` は **サーバー側の DB** に保存される。

### 3. リクエストごとに User を引き当てる

`app/controllers/application_controller.rb`

```ruby
def current_user
  @current_user ||= User.find_by(id: session[:user_id]) if session[:user_id]
end
```

ブラウザが cookie を送る → middleware が `sessions` テーブルから data を復元 → `session[:user_id]` が読める → `User.find_by` で User を取得。

`||=`（memoization）で **同一リクエスト内では DB を 1 回しか引かない**。

### 4. GraphQL context に詰めて resolver で使う

`app/controllers/graphql_controller.rb`

```ruby
context = {
  current_user: current_user,   # ApplicationController#current_user を呼び出し
  session: session,
  controller: self,
}
```

`app/graphql/resolvers/current_user.rb`

```ruby
def resolve
  context[:current_user]   # context から取り出すだけ
end
```

## どこに何が保存されるか

| 保存先 | 中身 |
|--------|------|
| ブラウザの cookie (`_app_session`) | `session_id`（ランダム文字列）のみ |
| MySQL の `sessions` テーブル | `{ session_id, data: 暗号化された {user_id: 42} }` |
| MySQL の `users` テーブル | ユーザー本体（メアド・パスワードハッシュ・名前など） |

## よくある誤解との比較

| 方式 | cookie の中身 | サーバー側の状態 | このプロジェクトは？ |
|------|---------------|-----------------|-----------|
| JWT 認証 | 署名付き JSON（user_id を含む） | なし（ステートレス） | ❌ |
| Cookie Store | 暗号化された JSON（user_id を含む） | なし | ❌ |
| **DB Session Store** | **session_id のみ** | **sessions テーブル** | **✅ これ** |

## 用語整理

| 用語 | 意味 |
|------|------|
| `session`（ハッシュ） | Rails が提供する「リクエスト間でデータを保持するための入れ物」 |
| `session[:user_id]` | そのハッシュに `:user_id` というキーで保存した値（= 42 など） |
| `session_id` | sessions テーブルの主キーになるランダム文字列（= "abc123..."） |
| `_app_session` | cookie の名前。中身は `session_id` |

## ポイント

- DB ベースのセッションは、cookie には `session_id` のみを入れ、`user_id` などの実データは **サーバー側の MySQL `sessions` テーブル**に保存する。JWT / Cookie Store とはここが本質的に違う。
- Rails の API mode ではセッション機能がデフォルト無効なので、`ActionDispatch::Cookies` と `ActionDispatch::Session::ActiveRecordStore` の middleware を手動で追加する。
- ログイン処理は `session[:user_id] = user.id` の 1 行で完結する。実際の DB 書き込みと `Set-Cookie` 付与は middleware がレスポンス時に自動で行う。
- `current_user` は `||=` でメモ化されており、同一リクエスト内では DB を 1 回しか引かない。
- GraphQL では `current_user` を context に詰めて resolver から `context[:current_user]` で参照する。
- 用語の区別: `session`（ハッシュ）/ `session[:user_id]`（値）/ `session_id`（cookie に入るランダム文字列）/ cookie 名（`_app_session`）の4つを混同しないこと。
