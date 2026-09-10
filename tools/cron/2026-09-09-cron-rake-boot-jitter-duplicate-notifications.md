---
title: 毎分 cron の時刻窓は Time.current ではなく分単位切り捨ての半開区間で決める
date: 2026-09-09
tags: [cron, rails, whenever, rake, time-window]
source: 実務(Rails)
---

# 毎分 cron の時刻窓は Time.current ではなく分単位切り捨ての半開区間で決める

## 概要

whenever gem による毎分 cron の rake タスクで、リマインダー通知の対象を「Time.current ± 30秒（両端含む）」の時刻窓で絞り込んでいたところ、同じ予約への通知がたまに二重送信された。rake タスクは実行のたびにフル Rails ブートを挟むため、Time.current には数秒〜数十秒のジッターが乗り、連続する実行の窓が重なったり隙間ができたりする。窓の基準を `beginning_of_minute` で分単位に切り捨て、幅1分の半開区間 `[t, t + 1.minute)` にすることで、窓が決定的に定まり重複も欠落もなくなった。

## サンプルコード

```ruby
# 「実行時刻の分単位切り捨て」を起点に、幅1分の半開区間で絞り込む
base = Time.current.beginning_of_minute + REMIND_BEFORE_MINUTES.minutes
Reservation.where("starts_at >= ? AND starts_at < ?", base, base + 1.minute)
```

## ポイント

- 定期実行の対象絞り込みで「実行された瞬間の現在時刻」を信用しない。cron 発火から実際のコード実行までにはブート時間などのジッターが乗り、窓の位置が実行タイミング依存で非決定的になる。前回が +40秒・今回が +20秒なら窓が重なって二重通知、逆方向にずれれば隙間ができて通知漏れになる
- 両端 inclusive（`>=` と `<=`）の窓は隣接する窓と端を共有するため、境界ちょうどのレコードはジッターが無くても2回拾われ得る。時刻範囲は半開区間 `[t, t + interval)` にする
- `beginning_of_minute` で切り捨てれば「同じ分の実行なら同じ窓」になり決定的。半開区間と組み合わせると隣接する分の窓と重複も隙間もなく、境界レコードもどちらか一方にだけ入る
- 前提として、窓幅は cron の実行間隔（この場合1分）と一致させること
- 回帰テストは `ActiveSupport::Testing::TimeHelpers` の `travel_to` で「1回目は +40秒、2回目は +80秒」というジッター付きの連続実行を再現し、同じレコードが片方の窓でのみ拾われることを検証する（旧実装ではこのテストが失敗する）。副次の学びとして、同一デバイストークンへの重複送信の排除は `next unless sent_tokens.add?(token)` の1行で書ける（`Set#add?` は追加できたら self、既存なら nil を返す）
