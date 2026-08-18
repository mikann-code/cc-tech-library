---
title: 毎分 cron の「時刻窓」で対象を拾うジョブは、起動の揺らぎで必ず二重実行しうる
date: 2026-08-18
tags: [cron, whenever, rake, idempotency, race-condition]
source: 実務(Rails)
---

## 概要

Rails アプリで、指定時刻の少し前に通知を送るバッチを cron / whenever / rake の3層で定時実行していた。cron は1分刻みでしか起動できないのに対し対象データの時刻は秒単位の任意の値を取るため、「N分後 ± 30秒」という時刻窓で対象を抽出していたところ、同じ相手に同じ通知が2回届いた（毎回ではなく特定の時間帯に偏って発生）。窓の中心がタスク内の `Time.current` で決まるため窓が前後にずれる、というのが原因で、tolerance の調整では直らず UNIQUE 制約による冪等化で解いた。

3層の役割はきれいに分かれている。cron は OS の定時実行（`分 時 日 月 曜日 コマンド`。`* * * * *` なら毎分、`0 19 * * *` なら毎日19時）で、保証するのは「その時刻にコマンドを起動する」ことだけ。whenever は crontab を Ruby の DSL で書ける gem で、変換するだけなのでサーバ上で `whenever --update-crontab` を実行して初めて反映される。rake は `lib/tasks/*.rake` にタスクを定義するタスクランナーで、`task foo: [:environment]` の `:environment` が Rails アプリ全体を読み込む依存タスク（これが無いとモデルも DB 接続も使えない）。

## サンプルコード

whenever の DSL と、そこから生成される crontab 行。

```ruby
# config/schedule.rb
every 1.minute do
  rake "notifications:deliver_upcoming"
end
```

```
* * * * * /bin/bash -l -c 'cd /app && RAILS_ENV=production bundle exec rake notifications:deliver_upcoming --silent'
```

冪等化の実装。外してはいけない点が2つある。①「先に SELECT で存在確認 → 無ければ INSERT」にしない（2プロセスが同時に「無い」と判断してすり抜ける TOCTOU 競合）。確認せず INSERT しにいって一意制約違反の例外を握る。②取り消せない副作用（外部への送信など）は最後に置き、記録を作れたときだけ実行する。

```rb
# db/migrate/xxxx_create_delivery_logs.rb
class CreateDeliveryLogs < ActiveRecord::Migration[7.0]
  def change
    create_table :delivery_logs do |t|
      t.references :reminder, null: false
      t.timestamps
    end
    add_index :delivery_logs, :reminder_id, unique: true
  end
end

# lib/tasks/notifications.rake
namespace :notifications do
  task deliver_upcoming: [:environment] do
    target_reminders.each do |reminder|
      next unless claim(reminder)  # 記録を作れたときだけ先へ進む
      Notifier.deliver(reminder)   # 取り消せない副作用は最後
    end
  end

  # 確認せず INSERT し、一意制約違反なら false（＝他が既に確保済み）
  def claim(reminder)
    DeliveryLog.create!(reminder: reminder)
    true
  rescue ActiveRecord::RecordNotUnique
    false
  end
end
```

この順序の代償として、記録は作れたが副作用の実行に失敗した場合は再実行されない。つまりこれは「重複実行」と「取りこぼし」のどちらのリスクを取るかの選択であって、両方を同時に完全解決するのは分散システムでは原理的に難しい。今回は「二重に届くより届かないほうがまし」という判断でこの順序を選んだ。

## ポイント

- **窓の中心は「cron が起動した時刻」ではなく「タスク内で `Time.current` を評価した時刻」で決まる。** `:environment` による Rails 読み込みが挟まるため、起動から時刻評価までの遅れ d は毎回ぶれる。N 回目の窓は `[起動時刻N + d(N) + オフセット - tolerance, 起動時刻N + d(N) + オフセット + tolerance]` になり、起動時刻は 60 秒ずつきっちり進むのに d は毎回違う。
- **二重実行と取りこぼしは同じ原因の裏表**で、片方だけを潰すことはできない。d が前回より小さければ窓が前にずれて前回と重なり（重なり幅は `d(前回) - d(今回)`）、大きければ後ろにずれて隙間ができる。したがって重なるか隙間が空くかは遅延の大小ではなく「前回との差」で決まり、常に3秒遅い環境では何も起きず、2秒と4秒を行き来する環境で問題が出る（安定した環境で再現しない理由）。
- **境界の等号は独立したバグ。** 窓の両端を `>=` と `<=` の等号込みで比較していると、d が完全に一定でも境界ちょうどのデータは前後どちらの窓にも入る。「毎時00分ちょうど」のようなキリのいい時刻に対象が集中すると境界付近に居座り続けるため、一様な低確率ではなく特定の時間帯に偏って繰り返し起きる。範囲比較は原則として片側を開区間にする。
- **「重複しない」を処理の順序やタイミングで担保しようとしない。** tolerance を詰めても窓幅を変えても、起動時間のぶれ自体は無くせないので直らない。一意性は DB の制約に持たせ、コードは制約違反を受け取って分岐するだけにする。SELECT してから INSERT する形は、単体では正しく見えて並行時に壊れる典型。
- **副作用の順序は「取り消せるものを先、取り消せないものを後」。** DB の記録は取り消せるが外部に出た副作用は取り消せない。これは cron に限らず、外部 API 呼び出しを含むあらゆる処理に効く原則。

## 参考

- [whenever — GitHub](https://github.com/javan/whenever)
- [Active Record Migrations — Rails Guides](https://guides.rubyonrails.org/active_record_migrations.html)
- [Command Line Tools / rake tasks — Rails Guides](https://guides.rubyonrails.org/command_line.html)
