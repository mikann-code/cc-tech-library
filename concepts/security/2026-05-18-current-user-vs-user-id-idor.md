---
title: currentUser でデータ取得すべき理由（userId 指定の脆弱性）
date: 2026-05-18
tags: [security, idor, authentication, authorization, graphql]
---

## 概要

クライアントから渡された userId で他人のデータを取得・更新できる API は、IDOR（Insecure Direct Object Reference）脆弱性を持つ。ログイン中のユーザー本人のデータを扱う操作では、userId を引数で受け取らず、サーバー側でセッションから current_user を解決して使うのが原則。

## サンプルコード

### NG パターン: userId をクライアントから受け取る

```graphql
mutation UpdateProfile($userId: ID!, $name: String!) {
  updateProfile(userId: $userId, name: $name) { ... }
}
```

```ruby
# resolver
def resolve(user_id:, name:)
  user = User.find(user_id)
  user.update!(name: name)
end
```

攻撃シナリオ:

- 攻撃者は自分のセッションでログインする
- ブラウザの DevTools / Burp / curl で userId を 他人の ID に書き換えてリクエストを送る
- サーバーは userId を信頼してしまうため、他人のプロフィールを書き換え／個人情報を取得できる

セッション（誰がログインしているか）と、操作対象（誰のデータを触るか）が乖離しているのが根本原因。

### OK パターン: サーバー側で current_user を解決する

```graphql
mutation UpdateProfile($name: String!) {
  updateProfile(name: $name) { ... }
}
```

```ruby
# resolver
def resolve(name:)
  current_user = context[:current_user]
  raise GraphQL::ExecutionError, "未ログイン" unless current_user

  current_user.update!(name: name)
end
```

## ポイント

- current_user はセッション cookie からサーバーだけが復元する。クライアントは「自分の ID」を渡す必要も、知る必要もない。リクエスト改ざんで他人を指定する余地がそもそも存在しない。
- 信用してよい場所はサーバー側のセッション（cookie の署名付き値、サーバー DB の sessions テーブル）。信用してはいけない場所はリクエストボディ、URL パラメータ、GraphQL 変数、ヘッダー（Authorization 等を除く）。
- クライアント側のコードや UI は、攻撃者にとって全部書き換え可能。「フロントが送ってこないから大丈夫」は防御にならない。サーバーが受け取った値はすべて疑うのが基本姿勢。
- 主催者が参加者を管理する、管理者が任意ユーザーを編集する、といった正規に他人を指定する API は別の話。この場合はクライアントから対象 ID（participant_id 等）を受け取り、サーバー側で current_user の権限をチェック（例: その participant が主催する施設のイベントに申込んでいるか）、OK なら操作を実行する。
- 自分のデータを操作 → userId を受け取らない（current_user だけ使う）。他人のデータを操作 → 対象 ID を受け取り、current_user の権限を必ず検証する。
