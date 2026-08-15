---
title: FCM プッシュを唯一の通知経路にすると「送れなかった」ことすら分からなくなる
date: 2026-08-12
tags: [fcm, push-notification, rails, cron, reliability]
source: 実務(Rails)
---

## 概要

モバイルアプリ向け Rails バックエンドで、予定のリマインダー（開始 60 分前 / 10 分前）を FCM プッシュだけで送る実装を読んだ記録。毎分起動する cron が「目標時刻の前後 30 秒の窓に入る予定」を拾って端末トークンへ順に送る作りで、コードは動くし例外も出ないのに「通知が来ていない」が起きうる形になっていた。しかも起きたことがサーバー側の記録に一切残らない。

（確認できたのはサーバー側の送信実装だけなので、APNs 証明書の設定や iOS 側の通知許可フロー、`UNUserNotificationCenter` の扱いには触れない。）

## サンプルコード

Before は元の構造をそのまま縮めたもの。空なら無音、1 件の例外で全員巻き添え、記録なし。

```rb
def deliver_reminders(users)
  users.each do |user|
    user.devices.each do |device|
      Notification.push(device.token, "start at #{user.item.starts_at.hour}:#{user.item.starts_at.min}")
    end
  end
end
```

After。効くのは「1 件ずつ隔離する」「結果を残す」「冪等にする」の 3 点。

```rb
def deliver_reminders(users)
  users.each do |user|
    devices = user.devices
    if devices.empty?
      Rails.logger.warn("no device token: user=#{user.id}")  # 無音にしない
      next
    end

    devices.each do |device|
      # 送信済みなら二重送信しない（窓を跨いだ再実行でも安全）
      next if Notification.sent?(user: user, item: user.item, kind: :reminder)

      begin
        Notification.push(device.token, "start at #{format('%02d:%02d', ...)}")
        Notification.record_sent(user: user, item: user.item, kind: :reminder)
      rescue Notification::InvalidToken
        device.destroy          # ローテーション済みトークンを掃除
      rescue => e
        Rails.logger.error(e)   # 1件の失敗をここで止め、残りを続行
      end
    end
  end
end
```

あわせて、コード外で効くもの:

- 管理側の画面に「通知先未登録」を表示し、届いていないことを人間が検知できるようにする
- 重要な通知はメールなど別経路も併用する（プッシュ 1 本にしない）
- 窓で拾う設計をやめるか、送信済みを永続化したうえで「未送信かつ時刻を過ぎたもの」を次の実行で拾い直す（catch-up）

## ポイント

- 原因は **「送信 API を呼べた ＝ 通知が届いた」という前提で全体が組まれていたこと**。前提として FCM はベストエフォート配信で、OS レベルの通知オフ、Android の Doze モード、iOS の集中モード、機内モードなどサーバーから制御できない要因で届かない。「送った」と「届いた」は別物という一線がコードに引かれていなかった。
- **送り先が存在しないケースが失敗として扱われていない。** トークンの配列が空なら `each` は 0 回まわって正常終了する。通知許可を拒否したユーザーや機種変更後に再登録していないユーザーがこれに該当するが、コードからは「エラーなし」と区別がつかず、管理画面にも表示が無いので人間が気づく経路も無い。
- **1 件の失敗が全体を巻き込む。** `rescue` の無い `each` の中で外部 API を叩いている。FCM のデバイストークンは端末側で自動的にローテーションするので無効トークンは通常運用で発生し、掃除する処理も無いため、腐ったトークンが 1 件残ると毎回同じ場所でバッチが落ち続ける。
- **時刻窓の設計に冪等性も catch-up も無い。** 「±30 秒の窓に入るものだけ送る」はその 1 分間の実行が成功することに全面依存する。実行が長引く / デプロイ中で止まるなどで窓を跨げばその通知は永久に送られず、逆に窓の中で 2 回走れば二重に飛ぶ。送信済みフラグも履歴も残していないので拾い直しも重複排除もできない。
- 結論として、**通知は「送信処理」ではなく「配信の状態機械」として設計する。** 未送信 / 送信済み / 失敗 がどこにも残らないコードは、失敗を検知できないだけでなくリトライも重複排除も原理的に不可能。チェックリスト: ①送り先 0 件は正常終了か ②外部 API を叩くループに `rescue` はあるか ③外部発行のトークン・ID は必ず腐る前提か ④2 回走ったら二重に飛ぶか / 0 回なら誰が拾い直すか ⑤届かなかったときユーザーに他の気づき方があるか。とくに ③④ は FCM に限らず外部プロバイダへ投げる非同期処理すべてに効く。

## 参考

- Firebase Cloud Messaging 公式ドキュメント: https://firebase.google.com/docs/cloud-messaging
