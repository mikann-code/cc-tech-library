---
title: GraphiQL 初学者ガイド
date: 2026-06-27
tags: [graphiql, graphql, ide, playground, rails]
---

## 概要

GraphQL API をブラウザ上で手動で叩いて試す IDE / プレイグラウンドである GraphiQL の初学者向けガイド。何ができるか、画面構成、基本操作、つまずきポイントをまとめる。

## サンプルコード

まず試す操作（Query で取得、Mutation で変更・実行）:

```graphql
# Query（取得）
{
  health
}

# Mutation（変更・実行）
mutation {
  noop
}
```

実行は Ctrl + Enter（Mac は Cmd + Enter）。整形は Prettify ボタン。

変数（Variables）の使い方。クエリ側で `$変数名` を宣言し、下の Variables パネルに JSON で渡す:

```graphql
query GetWordbook($uuid: String!) {
  wordbook(uuid: $uuid) { title }
}
```

```json
{ "uuid": "abc-123" }
```

認証ヘッダ（今後）。認証が必要なクエリは Headers に付与する想定:

```json
{ "Authorization": "Bearer <token>" }
```

## ポイント

- GraphiQL とは: GraphQL API をブラウザ上で手動で叩いて試す IDE / プレイグラウンド。"graphical + GraphQL" の造語。クエリの入力補完・スキーマのドキュメント参照・実行と結果表示ができる。手動の動作確認ツール（≠ 自動テスト）で、Postman の GraphQL 版に近い。
- このプロジェクトでの開き方: 開発サーバー起動後、http://localhost:3100/graphiql 。開発環境専用（routes.rb で `Rails.env.development?` のときだけマウント。本番では出ない＝安全）。実体は graphiql_controller.rb が CDN から GraphiQL を読み込む HTML を返しているだけ。
- 画面の構成: 中央（左）のクエリエディタで Query / Mutation を書く。下部 Variables にクエリ変数を JSON で渡す。下部 Headers に Authorization などのヘッダを付ける。▶ 実行ボタンでクエリ送信（/graphql に POST）。右の Docs / Schema でスキーマ（型・フィールド）を探索。
- 認証ヘッダ: 本プロジェクトでは現状トークン検証は未実装（認証 Issue #5/#6 で実装）。基盤としては context に current_user / current_admin を載せる経路だけ用意済み。
- Docs Explorer（最重要スキル）: 右上の Docs / Schema を開くと、その API にどんな Query / Mutation / 型があるかを辿れる。「何が叩けるか分からない」ときは、まずここを見るのが基本。スキーマが自己ドキュメントになっているのが GraphQL の強み。
- つまずきポイント:
  - GraphQL ≠ GraphiQL: 前者は API/言語、後者は試すツール。
  - GET では叩けない: API は POST /graphql。/graphiql は画面、/graphql が本体。
  - バージョン固定が必要: CDN を無指定にすると最新メジャー（React 19 / GraphiQL 4・5）になり UMD ビルドが無く壊れる → このプロジェクトは react@18 / graphiql@3 に固定済み（graphiql_controller.rb のコメント参照）。
  - 本番には無い: 開発専用なので、本番疎通の確認には使わない。
