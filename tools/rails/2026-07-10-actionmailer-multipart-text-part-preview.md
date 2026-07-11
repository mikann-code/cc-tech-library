---
title: multipart メールのテキスト版を読める形で確認する（decoded と ActionMailer::Preview）
date: 2026-07-10
tags: [rails, action-mailer, email, preview, multipart]
source: 実務(Rails)
---

## 概要

HTML/テキスト両方を送る `multipart/alternative` メールのテキストパートを、送信せずに人間が読める形で目視する2つの方法。メールクライアントは基本 HTML 版を優先表示し、生 MIME のテキスト版は `quoted-printable` エンコードで日本語が化けて読めないため、テキスト版の内容確認には別の手段が要る。

## サンプルコード

### 解決策1: 単発でデコードして目視（コンソール / runner）

メーラーを呼ぶと Mail オブジェクトが返る。`.deliver_now` を呼ばない限り送信されないので、安全に中身をデコードして確認できる。multipart メールは `text_part` / `html_part` で各パートを取り出せる。

```ruby
mail = SomeMailer.some_action(some_arg)

# テキスト版（quoted-printable をデコードして読める形に）
puts mail.text_part.body.decoded

# HTML 版
puts mail.html_part.body.decoded
```

### 解決策2: ブラウザで継続的に目視（ActionMailer::Preview）

Rails 標準の `ActionMailer::Preview` を使うと、開発環境で `http://localhost:3000/rails/mailers` から各メールをブラウザで開ける。画面上部の「View as: HTML / Plain text」トグルで両方をレイアウト適用済みで目視でき、送信もされない。プレビュー機能（画面・ルーティング・切替）は Rails に最初から組み込まれており、自分で用意するのはプレビュークラスだけ。

```ruby
class SomeMailerPreview < ActionMailer::Preview
  def some_action
    user = User.new(email: "sample@example.com")
    # DB に保存したくない属性はメモリ上で代入するだけで渡せる（保存不要）
    user.confirmation_token = "sample-token"

    SomeMailer.some_action(user)
  end
end
```

RSpec プロジェクトでプレビュークラスを `spec/` 配下に置きたい場合は、`config/environments/development.rb` に配置先を追加する（設定変更後はサーバ再起動が必要）。デフォルト配置先は `test/mailers/previews`。

```ruby
# config/environments/development.rb
config.action_mailer.preview_paths << Rails.root.join("spec/mailers/previews").to_s
```

## ポイント

- multipart メールは `text_part` / `html_part` でパート単位に取り出せる。`.body.decoded` で quoted-printable をデコードして読める形にできる。
- メーラーを呼ぶだけでは送信されない（`.deliver_now` を呼んで初めて送信）。安全にオブジェクトとして中身を検証できる。
- プレビューは開発環境限定（`show_previews` がデフォルトで本番 false なので本番では露出しない）。
- プレビュー用サンプルデータで、DB に保存したくないトークン等の属性は、レコードをメモリ上で属性代入するだけで渡せる。
- 単発チェックは解決策1、継続的な目視レビューは解決策2、と用途で使い分ける。

## 参考

- Rails Guides「Action Mailer Basics」Previewing Emails: https://guides.rubyonrails.org/action_mailer_basics.html#previewing-emails
