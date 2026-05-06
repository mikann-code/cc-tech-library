---
title: Apollo Client の使われ方（あるプロジェクトの方針）
date: 2026-05-06
tags: [apollo-client, graphql, nextjs, codegen, authentication]
---

## クライアント定義

- ファイル: `src/lib/apollo.ts`
- `HttpLink` で `${API_URL}/graphql` に接続
- `credentials: "include"` でクッキーセッション（`_app_session`）をクロスオリジン送信
- `InMemoryCache` を持つがキャッシュには依存しない設計

## Provider

- ファイル: `src/providers/ApolloProvider.tsx`
- `layout.tsx` で最外層にラップ（`ApolloProvider` > `LocaleProvider` > `ThemeProvider`）

## クエリ・ミューテーションの書き方

- `gql` テンプレートを `.ts` に直接書かない — 必ず `.graphql` ファイルを `src/graphql/` 配下に作成
- `yarn codegen` で `generated/*.generated.ts` に型と React hooks を自動生成
- インポート例:

```ts
import { useHogeMutation } from "@/graphql/mutations/generated/hoge.generated";
```

- 生成型はそのまま使う（独自の変換関数は作らない）

## fetchPolicy のルール

- すべての `useXxxQuery` で `fetchPolicy: "network-only"` を指定する
- 理由: Apollo Cache の挙動に依存せず、常に API から最新データを取得する方針

## ユーザー情報の管理（Cache を使わない理由）

- Apollo の正規化キャッシュではなく `AuthContext` を使う
- `(authenticated)/layout.tsx` で `useCurrentUserQuery()` を1回だけ実行 → `AuthContext.Provider` に格納
- 子コンポーネントは `useAuth()` で取得（`useCurrentUserQuery()` を直接呼ばない）
- 更新後は `refetch()` を呼んで再取得
- 設計意図: Apollo Cache の挙動変更に依存せず、データの流れと更新タイミングを明示的に制御するため。

## scalars マッピング

- `ISO8601DateTime` / `ISO8601Date` → `string`
- `Upload` → `File`
- `JSON` → `Record<string, unknown>`
