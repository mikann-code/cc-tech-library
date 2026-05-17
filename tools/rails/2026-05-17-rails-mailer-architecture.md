---
title: Rails のメール送信の仕組みとファイル構成
date: 2026-05-17
tags: [rails, mailer, action-mailer, solid-queue, architecture]
---

## 概要

Rails でメール送信を実装するときの登場人物（Mutation / Mailer / View / Layout / Solid Queue / Worker）と、それぞれのファイル構成、暗黙の規約による結びつき、非同期送信の流れ、責務の分離までを、メアド変更メールを例にまとめたもの。

## 1. 全体像

メアド変更のメール送信を例にすると、6つの登場人物 が協力して動いています。

```text
[ユーザー]
    │
    │ ① GraphQL リクエスト
    ▼
┌────────────────────────────────────────────┐
│ Mutation (RequestEmailChange)               │  ② 業務ロジック
│ - バリデーション                              │
│ - DB 更新                                    │
│ - メーラー呼び出し                            │
└──────────┬─────────────────────────────────┘
           │ ③ メーラーを呼ぶ
           ▼
┌────────────────────────────────────────────┐
│ Mailer (EmailChangeMailer)                  │  ④ メール組み立て
│ - 宛先・件名・変数セット                       │
└──────────┬─────────────────────────────────┘
           │ ⑤ テンプレートをレンダリング
           ▼
┌────────────────────────────────────────────┐
│ View (.html.erb / .text.erb)                │  ⑥ 本文の HTML/テキスト
│ ↓ レイアウトに埋め込まれる                     │
│ Layout (mailer.html.erb)                    │
└──────────┬─────────────────────────────────┘
           │ ⑦ ジョブとして登録
           ▼
┌────────────────────────────────────────────┐
│ Solid Queue (MySQL)                         │  ⑧ ジョブキュー
└──────────┬─────────────────────────────────┘
           │ ⑨ 後で取り出す
           ▼
┌────────────────────────────────────────────┐
│ Worker (panda_worker コンテナ)               │  ⑩ 実際に送信
│ ⇒ SMTP サーバーへ                            │
└──────────┬─────────────────────────────────┘
           │
           ▼
       [ユーザーのメールボックス]
```

## 2. ファイル構成

```text
packages/backend/
├── app/
│   ├── graphql/
│   │   └── mutations/
│   │       └── request_email_change.rb       ← ① 起点（リクエストを受ける）
│   │
│   ├── mailers/
│   │   ├── application_mailer.rb             ← ② 全メーラーの親（共通設定）
│   │   └── email_change_mailer.rb            ← ③ 個別のメーラー
│   │
│   ├── views/
│   │   ├── layouts/
│   │   │   ├── mailer.html.erb               ← ④ HTML メールの外枠
│   │   │   └── mailer.text.erb               ← ④ テキストメールの外枠
│   │   │
│   │   └── email_change_mailer/              ← ⑤ メール本文テンプレート
│   │       ├── email_change_email.html.erb       （HTML 版）
│   │       └── email_change_email.text.erb       （テキスト版）
│   │
│   └── jobs/
│       └── ...                               ← Solid Queue が裏で使う
│
├── config/
│   ├── initializers/
│   │   └── app_constants.rb                  ← SERVICE_NAME などの定数
│   └── queue.yml                             ← Solid Queue 設定
│
└── db/
    └── migrate/
        └── ..._create_solid_queue_*.rb       ← ジョブキュー用テーブル
```

## 3. 各ファイルの役割

### ① `request_email_change.rb`（Mutation = 起点）

役割: GraphQL リクエストを受けて、バリデーション・DB 更新・メール送信指示を行う 「司令塔」。

```ruby
def resolve(current_password:, email:)
  user = context[:current_user]

  # バリデーション
  errors = validate(...)
  return { success: false, errors: errors } if errors.any?

  # DB に pending_email とトークンを保存
  user.update!(
    pending_email: email,
    email_change_token: SecureRandom.urlsafe_base64(32),
    email_change_token_sent_at: Time.current,
  )

  # メーラーを呼んでメール送信を非同期で依頼
  EmailChangeMailer.email_change_email(user).deliver_later

  { success: true, errors: [] }
end
```

ここでやらないこと: メールの宛先・件名・本文の組み立ては Mailer に任せる（責務の分離）。

### ② `application_mailer.rb`（親メーラー = 共通設定）

役割: 全メーラーで共通の設定を1箇所にまとめる。

```ruby
class ApplicationMailer < ActionMailer::Base
  default from: ENV.fetch("MAILER_FROM", "noreply@example.com")  # 送信元
  layout "mailer"                                                  # 共通レイアウト
end
```

継承の恩恵: 子メーラーは送信元やレイアウトを毎回書かなくていい。

### ③ `email_change_mailer.rb`（個別メーラー = 組み立て係）

役割: メールの「宛先・件名・テンプレートに渡す変数」を準備する。

```ruby
class EmailChangeMailer < ApplicationMailer
  def email_change_email(user)
    @user = user
    @confirmation_url = "#{ENV.fetch('FRONTEND_URL', '...')}/.../confirm?token=#{user.email_change_token}"
    mail(to: user.pending_email, subject: "【#{SERVICE_NAME}】メールアドレス変更の確認")
  end
end
```

重要な設計: 宛先は `user.email` ではなく `user.pending_email`（新メアドの本人確認のため）。

### ④ `layouts/mailer.html.erb`（外枠 = レイアウト）

役割: 全メールに共通する HTML 構造を提供。

```erb
<!DOCTYPE html>
<html>
  <head>...</head>
  <body>
    <%= yield %>  ← ここに各メールの本文が埋め込まれる
  </body>
</html>
```

### ⑤ `email_change_email.html.erb`（本文 = テンプレート）

役割: 個別メールの本文を書く。

```erb
<h2><%= SERVICE_NAME %>のメールアドレス変更リクエストを受け付けました。</h2>
<p>以下のボタンをクリックして、メールアドレスの変更を完了してください。</p>
<p>
  <a href="<%= @confirmation_url %>" style="...">
    メールアドレスの変更を完了する
  </a>
</p>
```

メーラーで `@confirmation_url = ...` とセットした変数が、ここで `<%= @confirmation_url %>` として使える。

## 4. 暗黙の結びつき（Rails の規約）

このシステムは 明示的な設定がほとんどなく、ファイル名・メソッド名の規約で自動的に繋がっています。

| 結びつき | 規約 |
| --- | --- |
| Mailer クラス ↔ View フォルダ | `EmailChangeMailer` ↔ `views/email_change_mailer/` |
| Mailer メソッド ↔ View ファイル | `email_change_email` ↔ `email_change_email.html.erb` |
| `layout "mailer"` ↔ レイアウトファイル | `"mailer"` ↔ `layouts/mailer.html.erb` |
| Mailer の `@変数` ↔ View 内の参照 | `@confirmation_url` ↔ `<%= @confirmation_url %>` |

## 5. 非同期送信の仕組み

`deliver_later` がやっていること:

```ruby
EmailChangeMailer.email_change_email(user).deliver_later
```

| ステップ | 処理 |
| --- | --- |
| ① | `.email_change_email(user)` でメールを組み立てる |
| ② | `MessageDelivery` オブジェクトが返る（まだ送ってない） |
| ③ | `.deliver_later` で Solid Queue にジョブを登録 |
| ④ | API リクエストはここで完了 → ユーザーに即レス |
| ⑤ | `panda_worker` コンテナがジョブを取り出して 実際に送信 |

### deliver_now との違い

|  | deliver_now | deliver_later |
| --- | --- | --- |
| 動作 | その場で送信 | ジョブキューに登録 |
| レスポンス速度 | 遅い（数秒） | 速い（即時） |
| 障害耐性 | リクエスト失敗 | 自動リトライ |
| worker | 不要 | 必要 |

## 6. インフラ構成

### docker-compose のサービス

| サービス | 役割 | メール送信での出番 |
| --- | --- | --- |
| db | MySQL | ジョブキューの保管場所 |
| backend | Rails API | メール組み立て & ジョブ登録 |
| worker | Solid Queue ワーカー | 実際の送信実行 |
| frontend | Next.js | （直接関係なし） |

### Solid Queue が Redis の代わり

```text
[一般的な Rails]              [Panda]
Rails ↔ Redis ↔ Sidekiq       Rails ↔ MySQL ↔ Solid Queue
       （Redis 必要）                  （Redis 不要）
```

CLAUDE.md にある「Redis 不使用 — インフラをシンプルに保つ設計」という方針。

## 7. 環境変数

メール送信に関わる主な環境変数:

| 環境変数 | 用途 | 例 |
| --- | --- | --- |
| `MAILER_FROM` | 送信元アドレス | `noreply@panda.com` |
| `FRONTEND_URL` | メール内リンクのドメイン | `https://panda.example.com` |
| SMTP 設定 | メール送信先サーバー | （別途設定） |

開発環境では `.env` ファイルで管理、本番環境では AWS の環境変数で設定する想定。

## 8. メール送信フロー（時系列）

```text
時刻 t=0ms
[ユーザー] 「メアド変更」フォーム送信
    │
    ▼
[GraphQL] RequestEmailChange#resolve 実行開始

時刻 t=5ms
[Rails] バリデーション完了

時刻 t=15ms
[Rails] DB に pending_email & token 保存

時刻 t=20ms
[Rails] EmailChangeMailer.email_change_email(user) 実行
    │
    ├─ メーラー: @user, @confirmation_url をセット
    ├─ メーラー: テンプレート (.erb) をレンダリング
    ├─ レイアウト (mailer.html.erb) と合体
    └─ MessageDelivery オブジェクト生成

時刻 t=25ms
[Rails] .deliver_later で Solid Queue にジョブ登録
    │
    ▼
[MySQL] solid_queue_jobs テーブルにレコード追加

時刻 t=30ms
[Rails] GraphQL レスポンス返却 ✨ ユーザーに「メール送信しました」
                                  ─────────── ここでユーザーは解放 ───────────

時刻 t=数秒後（worker が次のポーリングタイミングで）
[worker] solid_queue_jobs からジョブ取得
    │
    ▼
[worker] EmailChangeMailer のメール送信実行
    │
    ▼
[SMTP] 外部メールサーバーへ送信
    │
    ▼
[ユーザーのメーラー] 受信
```

## 9. 責務の分離（まとめ）

各ファイルが 一つの責任に集中 している設計:

| ファイル | 責任 |
| --- | --- |
| Mutation | 「何をするか」の判断（バリデーション・DB 操作・メール送信の指示） |
| Mailer | 「メールをどう組み立てるか」の定義 |
| View (.erb) | 「メールの見た目」の定義 |
| Layout | 「全メールの共通枠」の定義 |
| Worker | 「実際に送る」の実行 |

これにより:

- メール本文を変えたい → View だけ修正
- 送信元を変えたい → `ApplicationMailer` の `default from:` だけ修正
- 新しい種類のメールを追加 → 新しい Mailer + View を作るだけ

という形で 変更が局所化 され、メンテナンスしやすい設計になっています。

## 10. 最後に

- ✅ メール送信は「組み立て」と「送信」が分離されている
- ✅ Mailer/View はファイル名規約で自動的につながる
- ✅ Layout で外枠を共通化し、各メールは本文だけ書けばいい
- ✅ `deliver_later` で非同期送信 → ユーザー体験が良い
- ✅ Solid Queue + MySQL で Redis なしの構成を実現

一言でいうと: 「Rails の規約に従ってファイルを置けば、リクエスト受付・組み立て・送信が自動的に連携する仕組みが出来上がる」 — これがメール送信機能の全体像です。
