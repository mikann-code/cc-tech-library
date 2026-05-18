---
title: 3段階トークンフロー — メール認証で本データを更新する設計パターン
date: 2026-05-18
tags: [security, authentication, design-pattern, email-verification, token]
---

## 概要

「メールアドレス変更」「パスワードリセット」「アカウント削除確認」のように、**操作の主体が本人であることをメール経由で確認してから本データを更新する**機能は、多くのアプリで必要になる。

教科書的には「mutation を1本叩いて更新」だが、実運用ではそれだとさまざまな事故が起きる。本章では「リクエスト → 検証 → 確定」の3段階に分割するパターンと、その各段階で何を達成すべきかを説明する。

## サンプルコード

### ナイーブな実装（問題あり）

```ruby
# ❌ ナイーブな実装
class UpdateEmail < BaseMutation
  argument :new_email, String, required: true

  def resolve(new_email:)
    user = context[:current_user]
    user.update!(email: new_email)
    { success: true }
  end
end
```

これだけでは以下の事故が起きる:

- **「宙ぶらりん状態」が発生する**: `email` カラムを直接書き換えると、旧メアドではもうログインできず、新メアドは本人が本当に所有しているか未検証になり、入力ミスで「自分のアカウントにログインできない」事故が起きる。
- **本人確認がない**: ログイン状態を維持したまま離席した端末を使われると、第三者が任意のメアドに書き換えてアカウントを乗っ取れる。
- **メーラーのプリフェッチで誤確定する**: 「メールにリンクを貼って、開いたら確定」という素朴なフローだと、Outlook / Gmail のセキュリティスキャナーや、URL プレビュー機能で**ユーザーがクリックする前に URL がアクセス**される。GET リクエストで状態が変わると、ここで誤確定する。
- **React Strict Mode の二重発火**: Next.js などの開発環境では `useEffect` が2回走る。確定処理を `useEffect` 内で叩くと2回実行される。

### 3段階フローの全体像

```
[Step 1] リクエスト（requestEmailChange）
  ・本人確認（現パスワードの照合）
  ・新メアドのバリデーション（形式・重複・現メアドとの同一性）
  ・トークン生成 + pending_email カラムに退避
  ・確認メール送信
  ↓ ユーザーがメール内リンクをクリック
[Step 2] 検証（confirmEmailChangeToken）
  ・トークンの有効性チェック（存在・期限）
  ・新メアドを画面に表示「○○ に変更しますか？」
  ・★ DB は変更しない（副作用なし）
  ↓ ユーザーが「変更を確定する」ボタンを押下
[Step 3] 確定（completeEmailChange）
  ・もう一度トークン検証
  ・email カラムを pending_email の値で書き換え
  ・pending_email / token をクリア
```

各段階の役割分担を明確に分けることで、それぞれの「失敗モード」を独立に防げる。

### Step 1: リクエスト（`request*`）

**やること:**
- 本人確認（パスワード再入力）
- 新値のバリデーション（形式・重複・「現在の値と同じ」拒否）
- ランダムトークン生成（`SecureRandom.urlsafe_base64(32)` 程度）
- `pending_*` カラムに新値を退避（本データはまだ書き換えない）
- 確認メール送信

```ruby
def resolve(current_password:, email:)
  user = context[:current_user]
  errors = validate(user:, current_password:, email:)
  return { success: false, errors: errors } if errors.any?

  # 本データ（user.email）は書き換えず、pending_email に退避
  user.update!(
    pending_email: email,
    email_change_token: SecureRandom.urlsafe_base64(32),
    email_change_token_sent_at: Time.current,
  )

  EmailChangeMailer.email_change_email(user).deliver_later
  { success: true, errors: [] }
end
```

DB スキーマには `pending_email`, `email_change_token`, `email_change_token_sent_at` の3カラムを追加する。

### Step 2: 検証（`*Token`）

**やること:**
- トークンの存在チェック
- 有効期限チェック（24時間など）
- 「○○ に変更しますか？」画面に表示する新メアドを返す
- **DB は一切変更しない**

```ruby
def resolve(token:)
  user = User.find_by(email_change_token: token)

  return error("無効なトークンです") if user.nil?
  return error("期限切れです") if user.email_change_token_sent_at < 24.hours.ago

  # ★ 副作用なし。検証結果と新メアドを返すだけ。
  { success: true, errors: [], email: user.pending_email }
end
```

**この副作用なし設計が重要**な理由:

| 副作用あり | 副作用なし |
|---|---|
| メーラーのプリフェッチで誤確定 | プリフェッチされても安全 |
| Strict Mode 二重発火で2回確定 | 何度呼ばれても結果同じ |
| リロードで誤確定 | リロード自由 |
| 多重タブで誤確定 | 多重タブ可 |

「検証用 mutation を副作用なしに保つ」ことが、3段階フローの肝。

### Step 3: 確定（`complete*`）

**やること:**
- もう一度トークン検証（Step 2 から時間が経っている可能性）
- 本データを `pending_*` の値で書き換え
- `pending_*` / トークンをクリア

```ruby
def resolve(token:)
  user = User.find_by(email_change_token: token)
  return error("無効なトークンです") if user.nil?
  return error("期限切れです") if user.email_change_token_sent_at < 24.hours.ago

  user.update!(
    email: user.pending_email,
    pending_email: nil,
    email_change_token: nil,
    email_change_token_sent_at: nil,
  )
  { success: true, errors: [] }
end
```

確定は **ユーザーの明示的なクリック**（フロントの「変更を確定する」ボタン）でのみ呼ぶ。`useEffect` で自動実行しない。

## ポイント

- 「リクエスト → 検証 → 確定」の3段階に分割し、それぞれの失敗モードを独立に防ぐ。
- リクエスト段階では本データ（`email`）を書き換えず、`pending_email` に退避して「宙ぶらりん状態」を防ぐ。
- 検証段階の mutation は**副作用なし**に保つ。これによりメーラーのプリフェッチ、Strict Mode 二重発火、リロード、多重タブのいずれでも誤確定しない。
- 確定段階はユーザーの明示的なクリックでのみ呼ぶ。`useEffect` で自動実行しない。
- リクエスト時は本人確認（現パスワード照合）を行い、ログイン状態を悪用したアカウント乗っ取りを防ぐ。
- トークン検証は Step 2 と Step 3 の両方で行う（Step 2 から時間が経っている可能性があるため）。
