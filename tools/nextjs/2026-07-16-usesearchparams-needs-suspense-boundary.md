---
title: App Router で useSearchParams() を使うページは Suspense 境界で包まないとビルドが落ちる
date: 2026-07-16
tags: [nextjs, app-router, suspense, usesearchparams, react]
source: 実務(Next.js)
---

## 概要

Next.js App Router で `useSearchParams()` を静的プリレンダリング対象ページで使うと、`next dev` では動くのに `next build` が `should be wrapped in a suspense boundary` で失敗する。クエリパラメータはビルド時に確定しないため CSR bailout が起き、fallback を与える Suspense 境界が必須になる。ページを「`<Suspense>` で Content を包むだけの薄い殻」にするのが定石。

## サンプルコード

```tsx
"use client";

import { Suspense } from "react";
import { useSearchParams } from "next/navigation";

function LoginContent() {
  const searchParams = useSearchParams();
  const redirect = searchParams.get("redirect");
  // ... フォーム本体
}

export default function LoginPage() {
  return (
    <Suspense fallback={<Spinner />}>
      <LoginContent />
    </Suspense>
  );
}
```

`?token=` を読むメール確認系ページなど、クエリパラメータを読むページはすべて同じ「薄い殻 + Content」構造に統一しておくと迷わない。

## ポイント

- 原因は、クエリパラメータがビルド時に確定しない値であること。`useSearchParams()` を使うコンポーネントはサーバー描画をそこで保留しクライアントで値を確定する（CSR bailout）ため、Next.js は fallback を与える Suspense 境界を必須にする。境界がないと保留がページ全体に波及するのでビルド時にエラー検出される。
- この形にするとビルドは成功し、ルートは静的（○）のままになる。ページ全体が動的化されるわけではなく、Suspense の内側だけがクライアントで確定する。
- Suspense が要る / 要らないの区別:

  | API / 機構 | Suspense 境界 | 理由 |
  | --- | --- | --- |
  | `useSearchParams()` | **必要** | クエリはビルド時に確定しない → CSR bailout |
  | `useParams()`（動的ルートセグメント） | 不要 | ルートの一部としてサーバー側でも確定している |
  | `usePathname()` | 不要 | 同上 |
  | `next/dynamic` | 不要 | `loading` オプション（fallback 機構）を内蔵 |
  | `React.lazy()` | 必要 | 遅延ロード中の fallback が要る |
  | React の `use(promise)` | 必要 | promise 解決まで描画を保留する |
  | Suspense 対応データ取得（例: `useSuspenseQuery`） | 必要 | 同上 |

- Apollo を非 Suspense の `useQuery`（`loading` フラグで分岐）で使う構成なら、実務で Suspense が必須になる契機は事実上 `useSearchParams()` だけ。「Suspense はデータ取得の話」と思っているとクエリを読んだだけでビルドが落ちて驚くので、この区別を覚えておく。
- 付随: `?redirect=` のような復帰先をクエリで受けるときは、外部 URL へ飛ばされないようオープンリダイレクト対策をしてから `router.push` する（「`/` 始まりかつ `//` 始まりでない」サイト内パスのみ許可）。

```ts
const isSafeRedirect = (path: string) =>
  path.startsWith("/") && !path.startsWith("//");
```

## 参考

- https://nextjs.org/docs/messages/missing-suspense-with-csr-bailout
- https://nextjs.org/docs/app/api-reference/functions/use-search-params
