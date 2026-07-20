---
title: Next.js の usePathname() はクエリ文字列を含まない — ログイン後リダイレクトで ?token=xxx が消える
date: 2026-07-17
tags: [nextjs, app-router, usePathname, auth-guard, open-redirect]
source: 実務(Next.js)
---

## 概要

Next.js（App Router）の認証ガードで「ログイン後に元のページへ復帰」を実装する際、`usePathname()` の値だけを redirect パラメータに渡すと、クエリ付き URL（メール確認リンク `/settings/email/confirm?token=xxx` 等）の `?token=xxx` が復帰先から欠落する。`usePathname()` はパスのみを返しクエリ文字列を含まないため。`window.location.search` を連結して解決する。

## サンプルコード

`window.location.search`（クエリがあれば `?key=value`、なければ空文字を返すブラウザ標準 API）を pathname に連結する。

```tsx
"use client";

import { useEffect } from "react";
import { usePathname, useRouter } from "next/navigation";

export default function AuthGuardLayout({ children }: { children: React.ReactNode }) {
  const router = useRouter();
  const pathname = usePathname();
  const isLoggedIn = useIsLoggedIn(); // 認証状態の取得は各自の実装で

  useEffect(() => {
    if (!isLoggedIn) {
      // usePathname() はクエリを含まないので window.location.search を連結する
      const returnTo = pathname + window.location.search;
      // replace: ガード発動ページを履歴に残さない（戻るボタン対策）
      router.replace(`/login?redirect=${encodeURIComponent(returnTo)}`);
    }
  }, [isLoggedIn, pathname, router]);

  return <>{children}</>;
}
```

- クエリなしページでは `window.location.search` が空文字なので挙動は変わらない。
- `useEffect` 内での使用なので、サーバーサイドで `window` が未定義になる問題はない。
- `useSearchParams()` でも可能だが、`.toString()` して `?` を付け直す組み立てが必要になるため、この場面では `window.location.search` の方が簡潔。

## ポイント

- `usePathname()` はクエリを含まない。「現在の URL」を復元したいときは search の連結が必要。
- redirect 復帰はページ個別ではなく認証境界（layout）の共通ガードに置くと、全認証ページが恩恵を受ける。
- redirect パラメータはオープンリダイレクト対策（`/` 始まりのみ許可・`//` 始まりは拒否＝サイト内絶対パスのみ許可）とセットで実装する。
- ガードのリダイレクトは `push` ではなく `replace`（戻るボタンでガードページに戻らないように）。
