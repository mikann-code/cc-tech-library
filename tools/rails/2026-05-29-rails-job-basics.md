---
title: Rails の Job まとめ（ActiveJob / Solid Queue / perform_later / recurring.yml）
date: 2026-05-29
tags: [rails, ruby, job, active-job, solid-queue, design]
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

## 追記：Job に責任を分離する 8 つのメリット

> 上記の基礎まとめ（構造・発動・実装テクニック・インフラ）を前提として、「**なぜ** Mutation / Controller から Job に切り出すのか」を 8 つの観点で深掘りする。題材は引き続き `NotifyEventApplicantsJob`。

### 概要

Rails の Mutation / Controller の中で重い処理を直接書かず、ActiveJob（このプロジェクトでは Solid Queue）に切り出すと何が嬉しいのかを、一斉メール送信ジョブを題材に整理する。

### メリットの全体像

| # | メリット | 何が嬉しいか |
|---|---|---|
| ① | レスポンス速度 | HTTP リクエストが即座に返る |
| ② | 失敗の隔離 | 1 件の失敗が他に波及しない |
| ③ | リトライ機能 | 一時的な失敗を自動で吸収 |
| ④ | スケーラビリティ | Worker を増やせば処理能力が線形に上がる |
| ⑤ | 責務の明確化 | Mutation・Job・Mailer が各々 1 つに集中 |
| ⑥ | 障害分離 | 外部サービス障害が本体に影響しない |
| ⑦ | 監視・運用の容易さ | キューの状態でシステム健全性が見える |
| ⑧ | テスタビリティ | 各レイヤーを独立して検証できる |

### ① レスポンス速度

#### Job がない場合（同期実行）

```ruby
def resolve(event_id:, target_status:, subject:, body:)
  # ...
  user_ids.each do |user_id|
    user = User.find(user_id)
    EventMessageMailer.notify(...).deliver_now   # 1 人 2 秒
  end
  # 100 人なら 200 秒待ち
end
```

```
主催者「送信ボタン」
   ↓（200 秒経過）
画面ようやく「成功」表示
HTTP リクエストがタイムアウト
主催者が「壊れた?」とボタン連打
途中で接続切れたら、状態が不明
```

#### Job がある場合（このプロジェクトの選択）

```ruby
def resolve(event_id:, target_status:, subject:, body:)
  # ...
  NotifyEventApplicantsJob.perform_later(...)   # 数ミリ秒
  { success: true, enqueued_count: user_ids.size }
end
```

```
主催者「送信ボタン」
   ↓（数百ミリ秒）
画面「30 件に送信を開始しました」 ← 即座
   ↓
裏で Worker がのんびり実行
```

| 観点 | 効果 |
|---|---|
| UX | 主催者が待たされない |
| インフラ | HTTP コネクションを長時間握らない |
| 信頼性 | 接続切れでも処理が継続 |

> 使い分けの判断軸は §5–6 を参照（メール 1 通 = `deliver_later` / 複雑な制御 = `perform_later`）。

### ② 失敗の隔離

#### Job がない場合

```ruby
user_ids.each do |user_id|
  EventMessageMailer.notify(...).deliver_now   # 1 人失敗 → 例外
end
```

→ 1 人目で失敗したら 2 人目以降が送られない。

#### Job がある場合（個別 `rescue` 設計）

```ruby
User.where(id: user_ids).find_each do |user|
  EventMessageMailer.notify(...).deliver_now
rescue StandardError => e
  # 1 件の送信失敗で全体を止めない
  Rails.logger.error("...")
end
```

→ 1 件失敗しても残り全員に送る。Ruby 2.5+ の `do...end` ブロック内 `rescue` 構文。

| 観点 | 効果 |
|---|---|
| 配信率 | 部分的にでも届く |
| 影響範囲 | 1 件の不正アドレスで全体停止しない |
| デバッグ | どの宛先で失敗したかログに残る |

> §7「パターン 2」§8-D「ループ内 `rescue` で失敗を隔離」の設計が、Job という「段取り専用の場所」があるからこそ書ける防衛ロジックになっていることが分かる。Mutation の中に書くとコードが太る。

#### 関連テクニック: ID 配列で渡す（§8-C と対応）

```ruby
# Mutation 側
NotifyEventApplicantsJob.perform_later(user_ids: [101, 205, 308])

# Job 側で再取得
User.where(id: user_ids).find_each(&:...)
```

- シリアライズが軽い（数値 vs ActiveRecord オブジェクト）
- Job 実行時に最新データを取れる
- データが消えていたら自然に弾ける（`find_by` + `return unless` のガードと組み合わせる）

→ 失敗の隔離は「ID 配列で渡す」「`find_each` で省メモリ」「ループ内 `rescue`」がワンセット。

### ③ リトライ機能

ActiveJob には自動リトライ機能がある:

```ruby
class NotifyEventApplicantsJob < ApplicationJob
  retry_on Net::SMTPError, attempts: 3, wait: 5.seconds
end
```

→ 失敗したら 5 秒後に最大 3 回リトライ。一時的な SMTP エラーをほぼ吸収できる。

#### このプロジェクトの現状

`application_job.rb`:

```ruby
class ApplicationJob < ActiveJob::Base
  # retry_on ActiveRecord::Deadlocked     ← コメントアウト
end
```

→ 今 `retry_on` を有効化していないが、いつでも有効化できる位置にある。§9「割り切り」表にもある通り、MVP 段階の意図的な省略。

#### Mutation 内に直接書いた場合

```ruby
def resolve(...)
  user_ids.each do |user_id|
    begin
      Mailer.notify(...).deliver_now
    rescue Net::SMTPError
      # ここでリトライ?
      # でも HTTP リクエスト中なのでタイムアウトする
    end
  end
end
```

→ HTTP リクエスト中のリトライは現実的でない。**Job だからこそリトライが意味を持つ**。

| 観点 | 効果 |
|---|---|
| 信頼性 | 一時的な障害を吸収 |
| 対応コスト | 開発者が手動で再実行しなくて済む |
| 可観測性 | 何回失敗したかが記録される |

### ④ スケーラビリティ

#### Job がない場合

メール送信が遅い → Web サーバを増やすしかない。でも Web サーバを増やすと:

- DB 接続数も増える
- メモリも食う
- HTTP リクエスト処理の本業に影響

#### Job がある場合

「Worker だけ増やせる」:

```yaml
# config/queue.yml
workers:
  - queues: "*"
    processes: 3   # ← 1 → 3 にするだけ
    threads: 3
```

→ メール送信速度は 3 倍に。Web サーバはそのまま。

```
Web サーバ ──HTTP リクエスト処理── ユーザー応答（速い）
   ↓ ジョブ投入
[キュー]
   ↓
Worker 1                Worker 2                Worker 3
（並列に処理 = スループット 3 倍）
```

| 観点 | 効果 |
|---|---|
| 本業への影響なし | Web サーバはユーザー対応に専念 |
| スケール方向の分離 | Web と Worker を独立にスケール |
| コスト効率 | 必要な部分だけ増強 |

> §10「インフラ構成（Solid Queue）」と対応。Solid Queue は MySQL ベースで Redis 不要、Worker は独立コンテナ（`app_worker`）として動く。中小規模なら十分なスループット。

### ⑤ 責務の明確化

#### Job なしの肥大化した Mutation

```ruby
def resolve(...)
  # 1. バリデーション
  # 2. 認可チェック
  # 3. 宛先抽出
  # 4. ユーザーループ                   ← Job の責務
  # 5. 1 人ずつメール組み立て            ← Mailer の責務
  # 6. SMTP 送信                         ← Mailer の責務
  # 7. 失敗時の rescue とログ            ← Job の責務
  # 8. 失敗時のリトライ判断              ← Job の責務
  # 9. レスポンス返却
end
```

→ Mutation が 9 つのことをやっている。テストも読解も困難。

#### Job ありの責務分離（このプロジェクト）

```
Mutation
├─ バリデーション
├─ 認可
├─ 宛先抽出
└─ ジョブ投入

Job
├─ ループ
├─ 失敗ハンドリング
└─ ログ記録

Mailer
└─ メール組み立て
```

| レイヤー | 責務 |
|---|---|
| Mutation | バリデーション + ジョブ投入 |
| Job | 段取り（ループ・エラーハンドリング・ログ） |
| Mailer | メール 1 通の中身（件名・本文・テンプレート） |
| SES | 実際の配送 |

| 観点 | 効果 |
|---|---|
| 可読性 | 1 ファイルで何が起きるかすぐ分かる |
| 変更容易性 | 特定の関心事だけ修正できる |
| テスト容易性 | レイヤー別にテスト可能 |

### ⑥ 障害分離

外部サービス（SES）が一時的にダウンしたら?

#### Job なしの場合

```
SES がダウン
   ↓
Mutation 内で SES 呼び出しが失敗
   ↓
HTTP リクエスト全体が失敗
   ↓
主催者「メール送信できません」エラー
   ↓
本来は別の操作もできるはずなのに、全体が止まる
```

#### Job ありの場合

```
SES がダウン
   ↓
Mutation はジョブを投入して即座に「成功」レスポンス
   ↓
（主催者は他の操作を続けられる）
   ↓
Worker がジョブを実行 → SES エラーで失敗
   ↓
リトライで吸収（数分後に SES 復活）
   ↓
事後的に送信完了
```

→ 本体（Mutation・主催者の操作）に SES 障害が伝わらない。

| 観点 | 効果 |
|---|---|
| 可用性 | 外部障害でアプリ全体が止まらない |
| ユーザー体験 | エラー画面を見せずに済む |
| 回復力 | リトライで自動回復 |

> Job がもたらす最大級のメリットの一つ。「不確実な処理の分離」シーンに該当。

### ⑦ 監視・運用の容易さ

Job があると、システムの状態をキュー経由で可視化できる:

```
Mission Control の管理画面
├─ 待機中ジョブ: 1,200 件 ← 異常に増えている → Worker 不足のサイン
├─ 失敗ジョブ: 50 件      ← SES 不具合のサイン
├─ 実行中ジョブ: 3 件
└─ 完了ジョブ: 45,000 件 ← 今日の処理量
```

→ 「システムが今何をしているか」が一目で分かる。Solid Queue は DB 永続化なのでクラッシュしてもジョブが消えず、状態を後から追える。

#### Job なしの場合

メール送信が遅い・失敗していても、ログを grep するか、ユーザーからの問い合わせを待つしかない。

| 観点 | 効果 |
|---|---|
| 可視化 | 処理状況が数値で見える |
| 早期発見 | 詰まりを管理画面で検知 |
| 再実行 | 失敗ジョブを手動で再キュー |

### ⑧ テスタビリティ

#### Job ありの分離テスト

```ruby
# Mutation のテスト: 「ジョブが投入されるか」だけ確認
RSpec.describe Mutations::NotifyEventApplicants do
  it "正常系でジョブが投入される" do
    expect {
      execute_graphql(...)
    }.to have_enqueued_job(NotifyEventApplicantsJob)
  end
end

# Job のテスト: 「ループとエラー処理」を確認
RSpec.describe NotifyEventApplicantsJob do
  it "宛先全員にメール送信を呼ぶ" do
    expect {
      described_class.perform_now(...)
    }.to change { ActionMailer::Base.deliveries.size }.by(3)
  end
end

# Mailer のテスト: 「メールの中身」を確認
RSpec.describe EventMessageMailer do
  it "件名にサービス名が付く" do
    mail = described_class.notify(...)
    expect(mail.subject).to start_with("【サービス名】")
  end
end
```

→ 各層が独立してテストできる。

#### Job なしの場合

```ruby
# Mutation のテストで全部やる
it "..." do
  # 入力検証も、ループも、メール内容も、エラー処理も
  # 全部 1 つのテストで…
end
```

→ テストが太く、何を保証しているか不明瞭。

| 観点 | 効果 |
|---|---|
| テスト設計 | 関心事ごとにテストを書ける |
| 回帰検出 | どこが壊れたかすぐ分かる |
| CI 高速化 | 影響範囲が小さいテストはスキップ判断可能 |

> 本番コードは `perform_later` で投入、テストは `perform_now` で同期実行して結果を検証する（§5）。

### このプロジェクトの例で見る恩恵

`notify_event_applicants_job.rb` のクラスコメントには、まさにこれらのメリットが言語化されている:

```
# Mutation 内で .deliver_later を呼ばず明示的に Job 化している理由:
#   - 宛先数が多い場合に SES 側のレート制限・失敗を 1 ジョブ内でリトライしやすい  ← ③④
#   - 個々の送信失敗を握りつぶさずログに残せる                                    ← ②⑦
#   - 将来テンプレート追加・配信履歴永続化等を行う際の拡張点として一元化できる    ← ⑤
```

実装した人も、「リトライ」「失敗の隔離」「責務集中」を理由として明確に意識していることが分かる。

### デメリットも知っておく

公平のために、Job 化のコストも:

| デメリット | 緩和策 |
|---|---|
| 実装が複雑になる | Rails の規約に従えば最小限 |
| デバッグが難しい（非同期だから） | ログをしっかり残す |
| インフラが増える（Worker） | Solid Queue で MySQL に集約（Redis 不要） |
| 送信完了が即時に分からない | 配信履歴テーブルで後追い（将来課題） |

→ このプロジェクトはこれらを割り切って運用している（§9「割り切りの設計」と対応）。

### 8 メリットとこのプロジェクトでの体現

| # | メリット | このプロジェクトでの体現 |
|---|---|---|
| 1 | レスポンス速度 | Mutation が即座に `enqueued_count` を返す |
| 2 | 失敗の隔離 | `rescue` で 1 件失敗を全体に波及させない |
| 3 | リトライ機能 | 将来 `retry_on` で吸収可能 |
| 4 | スケーラビリティ | `queue.yml` で Worker 数を変えられる |
| 5 | 責務の明確化 | Mutation / Job / Mailer が各々 1 つに集中 |
| 6 | 障害分離 | SES 障害が Mutation に影響しない |
| 7 | 監視・運用の容易さ | Mission Control でキュー可視化 |
| 8 | テスタビリティ | 各層を独立してテスト |

### 一行で言うと

> 「重い処理 / 不確実な処理 / 段取りが必要な処理」を本体から切り離し、専門の場所で扱うことで、システム全体の信頼性・可読性・運用性が一気に上がる

### 設計の感覚

```
本体 = 「お客様対応」  → 速く、正確に、シンプルに
Job  = 「裏方の仕事」  → 確実に、ねちっこく、リカバリ込みで
```

このプロジェクトの `NotifyEventApplicantsJob` は、まさにこの「本体は速く返して、裏方が確実に仕事する」を体現した好例。Job のないコード（Mutation で全部やる）と比較すると、信頼性・拡張性・テストしやすさのすべてで圧倒的に勝る。

「Mutation が薄くなって、Job が裏で確実に仕事する」という構造そのものが、Rails アプリのスケーラビリティと信頼性の根本を支えている、と理解すれば、なぜわざわざ分離するのかの納得感が深まる。

### 追記分のポイント

- Job 分離の本質は「重い処理 / 不確実な処理 / 段取りが必要な処理」を本体から切り離すこと
- 8 大メリットは **速度・隔離・リトライ・スケール・責務・障害分離・監視・テスト** の 8 軸
- 「失敗の隔離」は **ID 配列で渡す**・**`find_each` で省メモリ**・**ループ内 `rescue`** のセットで初めて成立する（§8 と対応）
- 「スケーラビリティ」は **Web と Worker を独立にスケールできる** こと。Solid Queue なら `queue.yml` の `processes` を変えるだけ
- 「障害分離」は外部サービス（SES など）の不安定さをアプリ本体から切り離す最大級の効果
- このプロジェクトは MVP 段階として `retry_on` や配信履歴を意図的に省略しているが、構造的にはいつでも追加できる位置にある
