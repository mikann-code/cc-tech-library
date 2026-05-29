---
title: Rails の Job まとめ（ActiveJob / Solid Queue / perform_later / recurring.yml）
date: 2026-05-29
tags: [rails, ruby, job, active-job, solid-queue]
---

## 概要

Rails の Job（ActiveJob + Solid Queue）の全体像を学んだまとめ。Job の発動方法・基本構造・典型パターン・実装テクニック・インフラ構成までを横断的に整理する。

## 1. Job とは何か

HTTP リクエストの中で完結させたくない処理を、別プロセスで動かすための仕組み。

### Job を使う典型シーン

| シーン | 例 | 理由 |
|---|---|---|
| 重い処理の非同期化 | 一斉メール送信 | リクエストを早く返したい |
| 定期実行（cron 的） | セッション掃除 | ユーザー操作と無関係に定期実行 |
| 失敗時のリトライ | 外部 API 呼び出し | 一時的失敗を吸収 |
| 不確実な処理の分離 | SES 送信 | 失敗を業務処理から切り離す |

## 2. このプロジェクトの Job 構成

```
packages/backend/app/jobs/
├── application_job.rb              ← 基底クラス（空に近い）
├── notify_event_applicants_job.rb  ← 一斉メール送信
├── cleanup_unattached_blobs_job.rb ← 不要ファイル掃除
├── publish_events_job.rb           ← イベント公開状態の更新
└── session_cleanup_job.rb          ← セッション掃除
```

### Job 分類マトリクス

| Job | 発動方法 | 起点 | 目的 |
|---|---|---|---|
| `NotifyEventApplicantsJob` | `perform_later` | 主催者の操作 | 重い処理の非同期化 |
| `SessionCleanupJob` | `recurring.yml` | 定期（毎日 4 時） | DB クリーンアップ |
| `CleanupUnattachedBlobsJob` | `recurring.yml` | 定期（毎日 5 時） | ストレージクリーンアップ |
| `PublishEventsJob` | `recurring.yml` | 定期（毎時） | ビジネスロジック更新 |

## 3. Job の基本構造

すべての Job が共通のパターン:

```ruby
class XxxJob < ApplicationJob   # ① 基底クラスを継承
  queue_as :default              # ② どのキューに入れるか

  def perform(...)               # ③ 実際の処理
    # ここに実行内容
  end
end
```

| 要素 | 必須? | 役割 |
|---|---|---|
| `< ApplicationJob` | ✅ | プロジェクト共通の基底 |
| `queue_as :default` | 任意 | 優先度・Worker 分離するときに使う |
| `def perform` | ✅ | 実際にやりたい処理 |

## 4. Job の発動方法（2 種類）

### 方法 A: コードから明示的に `perform_later`

```ruby
# Mutation や Controller から呼ぶ
NotifyEventApplicantsJob.perform_later(
  event_id: event.id,
  user_ids: user_ids,
  subject_text: subject,
  body_text: body
)
```

- ユーザー操作起点の非同期処理
- 例: メール送信、画像リサイズ、外部 API 呼び出し

### 方法 B: `recurring.yml` で定期実行

```yaml
# config/recurring.yml
session_cleanup:
  class: SessionCleanupJob
  schedule: at 4am every day

publish_scheduled_events:
  class: PublishEventsJob
  schedule: every hour
```

- 時間起点の定期実行（cron 的）
- 例: クリーンアップ、定期集計、状態更新

## 5. `perform_later` vs `perform_now`

| メソッド | 実行タイミング | 用途 |
|---|---|---|
| `perform_later` | 後で実行（キューに登録） | ほぼ常にこちら |
| `perform_now` | 今すぐ実行（同期） | テストや特殊用途 |

### メール送信との対応関係

| メール送信 | ジョブ実行 |
|---|---|
| `UserMailer.welcome.deliver_now` | `MyJob.perform_now` |
| `UserMailer.welcome.deliver_later` | `MyJob.perform_later` |

- `deliver_later` は内部的に `perform_later` を呼んでいる
- `perform_later` がより汎用の土台、`deliver_later` はメール特化のラッパー

## 6. 使い分けの判断フロー

```
非同期で処理したい
   ↓
それはメール 1 通の送信？
   ├── Yes → deliver_later（簡潔）
   └── No
        ↓
        メール以外 or 複雑な制御が必要？
              ↓
              perform_later（汎用・柔軟）
```

### 「メール送信でも `perform_later` を選ぶ」理由

このプロジェクトの `NotifyEventApplicantsJob` はメール送信ジョブだが、あえて Mailer 直接呼び出しではなく Job 経由:

- 宛先数が多い場合に SES 側のレート制限・失敗を 1 ジョブ内でリトライしやすい
- 個々の送信失敗を握りつぶさずログに残せる
- 将来テンプレート追加・配信履歴永続化等を行う際の拡張点として一元化できる

## 7. Job 実装の典型パターン

### パターン 1: 条件絞り込み + 一括処理

```ruby
# SessionCleanupJob
def perform
  Session.where("updated_at < ?", 30.days.ago).delete_all
end

# PublishEventsJob
def perform
  Event.scheduled
       .where("publish_at <= ?", Time.current)
       .update_all(status: "published")
end
```

→ シンプルな `where` + 一括操作。定期実行向き。

### パターン 2: ループ処理 + 個別エラーハンドリング

```ruby
# NotifyEventApplicantsJob
def perform(event_id:, user_ids:, subject_text:, body_text:)
  event = Event.find_by(id: event_id)
  return unless event   # ガード

  User.where(id: user_ids).find_each do |user|
    EventMessageMailer.notify(...).deliver_now
  rescue StandardError => e
    Rails.logger.error("...")
  end
end
```

→ 個別失敗を隔離しつつループ続行。重要な実務パターン。

### パターン 3: `find_each(&:purge)` 形式

```ruby
# CleanupUnattachedBlobsJob
def perform
  ActiveStorage::Blob.unattached
    .where("created_at < ?", RETENTION_DAYS.days.ago)
    .find_each(&:purge)
end
```

→ 取得しながら破壊的操作。クリーンアップ系で頻出。

## 8. Job 内部の重要テクニック

### A. `find_each` で省メモリ

```ruby
User.where(id: user_ids).find_each do |user|
  # 1000 件ずつバッチで取得・処理
end
```

| メソッド | メモリ |
|---|---|
| `.each` | 全件一気にメモリへ |
| `.find_each` | 1000 件ずつ（デフォルト）バッチ取得 |

大量データ処理の必須テクニック。

### B. `find_by` + `return` でガード

```ruby
event = Event.find_by(id: event_id)
return unless event   # 削除済みなら何もせず終了
```

| メソッド | 見つからないとき |
|---|---|
| `find` | 例外を投げる |
| `find_by` | `nil` を返す |

Job 実行時は Mutation 投入から時間が経っており、データが消えている可能性がある → `find_by` + nil ガードが安全。

### C. ID 配列を渡す（オブジェクトを渡さない）

```ruby
# ✅ Mutation 側
NotifyEventApplicantsJob.perform_later(user_ids: [101, 205, 308])

# Job 側で再取得
User.where(id: user_ids).find_each do |user|
  ...
end
```

理由:

- シリアライズが軽い（数値 vs ActiveRecord オブジェクト）
- Job 実行時に最新データを取れる
- データが消えていたら自然に弾ける

### D. ループ内 `rescue` で失敗を隔離

```ruby
User.where(id: user_ids).find_each do |user|
  EventMessageMailer.notify(...).deliver_now
rescue StandardError => e
  Rails.logger.error("...")  # ログを残して次のユーザーへ
end
```

Ruby 2.5+ の `do...end` ブロック内 `rescue` 構文。1 件の失敗で全体を止めない。

### E. Job 内では `deliver_now`

```ruby
EventMessageMailer.notify(...).deliver_now   # ← later ではない
```

Job 自体が既に非同期文脈なので、ここで `deliver_later` するとさらに別ジョブが作られて二重非同期になる。**Job 内では同期送信** が正しい。

## 9. このプロジェクトの Job 設計の特徴

### 責務分担

```
Mutation
   ↓ ジョブ投入（即座に応答）
Job
   ↓ メール生成依頼
Mailer
   ↓ 送信実行
SES
```

| レイヤー | 責務 |
|---|---|
| Mutation | バリデーション + ジョブ投入 |
| Job | 段取り（ループ・エラーハンドリング・ログ） |
| Mailer | メール 1 通の中身（件名・本文・テンプレート） |
| SES | 実際の配送 |

### 「割り切り」の設計

このプロジェクトの Job は以下を **意図的に省略**:

| 省略項目 | 代替 |
|---|---|
| 送信完了の主催者通知 | 投入件数 `enqueued_count` のみ返す |
| 失敗時のリトライ | ログだけ残して「諦める」 |
| 配信履歴の DB 永続化 | 将来の拡張点として明記 |
| バウンス取り込み | 運用ログ任せ |

→ MVP 段階としては妥当な割り切り。本格運用前にリトライ設定と配信履歴は追加すべき。

## 10. インフラ構成（Solid Queue）

| 項目 | このプロジェクトの選択 |
|---|---|
| ジョブキュー | Solid Queue（MySQL ベース） |
| Redis | 不使用 |
| Worker コンテナ | `app_worker`（独立プロセス） |
| 設定ファイル | `config/queue.yml`（キュー設定）、`config/recurring.yml`（定期実行） |
| 永続性 | DB トランザクションで保証（クラッシュしてもジョブが消えない） |

### Solid Queue の強み

- インフラがシンプル（MySQL だけ）
- ジョブ消失リスクが低い（DB 永続化）
- トランザクション整合性（DB の中で完結）

### 弱み

- Redis ベース（Sidekiq）より処理速度はやや劣る
- 大量ジョブのスループットは劣る

→ 中小規模なら問題なし。

## 11. 新しい Job を作るときの手順

### Step 1: ファイル作成

```bash
# 慣習: app/jobs/{snake_case}_job.rb
touch app/jobs/my_new_job.rb
```

### Step 2: クラス定義

```ruby
class MyNewJob < ApplicationJob
  queue_as :default

  def perform(arg1:, arg2:)
    # 処理を書く
  end
end
```

### Step 3: 発動方法を決める

ユーザー操作起点なら呼び出し元から:

```ruby
MyNewJob.perform_later(arg1: x, arg2: y)
```

定期実行なら `recurring.yml` に追加:

```yaml
my_new_job:
  class: MyNewJob
  schedule: every hour
```

### Step 4: テスト

```ruby
# spec/jobs/my_new_job_spec.rb
RSpec.describe MyNewJob do
  it "..." do
    # ...
  end
end
```

## 12. 全体まとめ

### Job の本質

「HTTP リクエストの中でやらない / やれない処理を別プロセスで動かす仕組み」

### キーポイント

| 観点 | 押さえどころ |
|---|---|
| 使い分け | メール 1 通 = `deliver_later` / それ以外 = `perform_later` |
| 発動 | コードから `perform_later` か `recurring.yml` で定期 |
| 基本構造 | `class XxxJob < ApplicationJob` + `def perform` |
| 設計の鉄則 | ID 配列で渡す、`find_each` で省メモリ、個別 `rescue` で隔離 |
| 責務分離 | Mutation = 投入、Job = 段取り、Mailer = メール中身 |
| 割り切り | リトライ・配信履歴は MVP では省略可、本番前に検討 |

### 用語の関係

```
ActiveJob（Rails のジョブフレームワーク）
   ↓ 抽象
Solid Queue（具体的なキュー実装。MySQL ベース）
   ↓ 設定
queue.yml / recurring.yml
   ↓ 実行
app_worker コンテナ
   ↓ 拾う
あなたの書いた XxxJob#perform
```

### このプロジェクトのシンプルさ

- 5 つの Job ファイル、合計 100 行以下
- Redis なし、cron なし、すべて MySQL で完結
- インストールしてすぐ動く、運用がシンプル

## 13. 参考: 関連ファイル

| ファイル | 役割 |
|---|---|
| `application_job.rb` | 全 Job の基底クラス |
| `notify_event_applicants_job.rb` | ユーザー操作起点の好例 |
| `publish_events_job.rb` | 定期実行の最小例 |
| `config/queue.yml` | Worker・キュー設定 |
| `config/recurring.yml` | 定期実行の cron 定義 |
| `docker-compose.yml` | `app_worker` サービス定義 |

## ポイント

- Job は「HTTP リクエストの中でやらない / やれない処理を別プロセスで動かす仕組み」
- 発動は 2 種類: コードから `perform_later`（ユーザー起点） / `recurring.yml`（時間起点）
- 設計の鉄則: **ID 配列で渡す**・**`find_each` で省メモリ**・**個別 `rescue` で隔離**・**Job 内では `deliver_now`**
- 責務分離: Mutation = 投入 / Job = 段取り / Mailer = メール中身 / SES = 実配送
- Solid Queue は MySQL ベースで、Redis 不要 → インフラがシンプルで中小規模なら十分
- 新しい Job を書くときは「ユーザー起点か、定期実行か」を判断して近い既存 Job をテンプレートにすると早い
