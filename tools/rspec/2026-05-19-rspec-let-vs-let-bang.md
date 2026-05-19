---
title: RSpec `let` と `let!` の違い・統一の重要度
date: 2026-05-19
tags: [rspec, ruby, testing, let, before-hook]
---

## 概要

RSpec の `let` と `let!` の違いは `!` の有無だけだが、各テストの開始タイミングで自動的にブロックを実行するかどうかが変わる。プロジェクト内で書き方を揃えると、`before { user }` の繰り返しが消えて読みやすくなる。

## サンプルコード

### 違いは `!` の有無だけ

| | `let(:user)` | `let!(:user)` |
|---|---|---|
| `User.create!` の実行タイミング | テスト内で `user` を参照したとき | 各テストの開始前に自動で |
| `before { user }` が必要か | はい | **いいえ** |

### `let!` は内部的に `let + before { user }`

```ruby
# let! の書き方
let!(:user) do
  User.create!(...)
end

# ↓ 内部的にはこう動いている
let(:user) do
  User.create!(...)
end
before { user }   # ← これが各テストの最初に勝手に走る
```

つまり `!` を付けるだけで、暗黙的に全テストの `before` に `user` 呼び出しが追加される。

### 変更前

```ruby
let(:user) do
  User.create!(...)
end

describe "正常なリセット" do
  before { user }      # ← 明示的に呼ぶ必要があった

  it "..." do
    # ここで user は DB にいる
  end
end

describe "存在しないトークン" do
  before { user }      # ← 同様

  it "..." do
    ...
  end
end
```

### 変更後

```ruby
let!(:user) do         # ← ! 1 文字追加
  User.create!(...)
end

describe "正常なリセット" do
  # before { user } 不要！
  it "..." do
    # それでも user は DB にいる
  end
end

describe "存在しないトークン" do
  it "..." do
    ...
  end
end
```

### 残した before

```ruby
describe "有効期限切れのトークン（2時間超過）" do
  before { user.update!(reset_password_sent_at: 3.hours.ago) }
  ...
end
```

これは `user` を呼ぶためではなく、**状態を変えるため**の before なので残す。

## 動作と見た目の関係

```
変更前
  let(:user) + 各 describe に before { user }
  → User.create! は各テスト前に走る

変更後
  let!(:user)
  → User.create! は各テスト前に走る   ← 結果は同じ
```

**テストの実行結果は変わらない**。違うのはコードの見た目のみ。

- `before { user }` の繰り返しが消えた（7 行削除）
- `let!` 1 つで「全テストで User を作る」が完結する
- プロジェクト全体の他の spec と書き方が揃った

## どっちがプロジェクトの慣習？

| ファイル | パターン |
|---|---|
| `logout_spec.rb` | `let!(:user)` |
| `request_password_reset_spec.rb` | `let!(:user)` |
| `reset_password_spec.rb`（修正前） | `let + before { user }` ← 浮いていた |

このプロジェクトは **`let!` で統一**されている。新規 spec を書くときは既存の書き方に揃える。

## この変更の重要度

| 観点 | 重要度 | 理由 |
|---|---|---|
| テストの動作 | ❌ 大事じゃない | どっちでも同じ結果になる |
| セキュリティ | ❌ 大事じゃない | spec の書き方なので本番影響なし |
| コードの読みやすさ | 🟡 まあまあ | 7 行ノイズが減る |
| プロジェクトの統一性 | 🟡 まあまあ | 他の spec と揃って読みやすくなる |


## ポイント

- `let` と `let!` の違いは `!` 1 文字だけ。`let!` は「テスト内で参照されたとき」ではなく「各テストの開始前」に自動でブロックを実行する。
- `let!(:user)` は内部的に `let(:user) + before { user }` と等価。`before { user }` を各 describe に書く繰り返しが消える。
- 「データを作るための before { user }」は `let!` 化で消せるが、「状態を変えるための before」は残す（例: `before { user.update!(...) }`）。
- 動作は変わらず、変わるのは見た目だけ。プロジェクト内の他の spec と書き方を揃える意義はあるが、機能・セキュリティ的にはほぼ無意味。
- 修正の優先度を分けて考える: バグ・セキュリティ（高） / 規約違反（中） / 書き方の統一感・コメント粒度（低）。
