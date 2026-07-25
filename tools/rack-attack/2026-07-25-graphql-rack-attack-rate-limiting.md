---
title: 単一エンドポイントの GraphQL を Rack::Attack でレート制限する（と、ボディ rewind の罠）
date: 2026-07-25
tags: [rack-attack, graphql, rate-limiting, rack, middleware]
source: 実務(Rails)
---

## 概要

Rails + GraphQL の API で、login のような特定 mutation にブルートフォース対策のレート制限を Rack::Attack でかけたときの話。REST なら `POST /login` のように URL パスで `throttle` 対象を絞れるが、GraphQL は query も mutation もすべて `POST /graphql` に集約されるためパスでは区別できない。そこでリクエストボディの `query` 文字列を覗いて対象を判別したところ、レート上限に達していない通常のリクエストまで mutation が実行されず結果が `null` になった。原因は、Rack ミドルウェアで読んだリクエストボディを巻き戻していなかったこと。

## サンプルコード

```ruby
# ミドルウェア内でボディを読むときは前後で rewind する
body = request.body
body.rewind
raw = body.read
body.rewind   # ← これが無いと後段が空ボディを読む
```

```ruby
# query 文字列に対象 mutation 名が含まれるかを単語境界一致で判定する
def graphql_mutation?(raw, name)
  json = JSON.parse(raw) rescue {}
  query = json["query"].to_s
  query.match?(/\b#{name}\b/)   # 例: name = "login"
end
```

## ポイント

- リクエストボディはストリームで、一度 `read` すると読み取り位置が末尾まで進む。Rack ミドルウェアはアプリ本体より手前で動くため、巻き戻さないと後段のフレームワークが空ボディを読み、GraphQL のパラメータパースが壊れて mutation が実行されないまま 200 が返る。読んだら `rewind` はほぼ定型句。
- 「URL パスで対象を絞る」は REST の前提。単一エンドポイントに集約するプロトコル（GraphQL に限らず RPC 的なもの全般）では、識別子はボディの中にあると考える。
- 判別は AST パースまでせず mutation 名の単語境界一致で「ゆるく」倒すのが実用的。`name(` のように括弧まで要求すると名前と括弧の間の空白やコメントですり抜けられる。対象外が偶然マッチしてもカウントが 1 増えるだけで実害は小さい、というトレードオフで割り切る。
- 実運用の設計ポイント: ボディ JSON は複数の throttle ルールから参照されるのでインスタンス変数にメモ化して JSON パースは 1 回だけにする／不正 JSON は `rescue` して素通しする（どうせ後段で 400 になる）／カウンタ保存先は `Rails.cache` 抽象に載るので Redis 必須ではない／IP をキーにする `throttle` とボディ由来の識別子（例: email）をキーにする `throttle` を併用すると、1 回線からの総当たりと IP を分散した単一対象狙いの両方に効く。
- テスト観点: rewind が効いていることは、レート上限内のリクエストで「後段の mutation が通常どおり実行され、意図したレスポンスを返す」ことをリクエストスペックで確認すると担保できる。
