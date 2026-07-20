---
title: 認証ガードのリダイレクトは router.push ではなく router.replace で書く
date: 2026-07-17
tags: [nextjs, app-router, history-api, auth-guard, ux]
source: 実務(Next.js)
---

## 概要

Next.js（App Router）で未ログインユーザーを `/login` へ飛ばす認証ガードを `router.push("/login")` で書くと、ログイン画面で「戻る」を押すとガード発動ページに戻り、そこで再びガードが発動して弾き返される——「戻るボタンが壊れている」体験になる。認証ガードは「システムが強制した遷移」なので、履歴を積む `push` ではなく現在エントリを置き換える `replace` で書き、復帰先は `?redirect=` で渡すのが定石。

## サンプルコード

ガードは `replace` に変え、あわせて復帰先を `redirect` パラメータで渡す。

```tsx
"use client";

import { useEffect } from "react";
import { usePathname, useRouter } from "next/navigation";

export function AuthGuard({ children }: { children: React.ReactNode }) {
  const router = useRouter();
  const pathname = usePathname();
  const { currentUser, loading } = useAuth(); // 認証状態の取得は各アプリの実装に依存

  useEffect(() => {
    if (!loading && !currentUser) {
      // push だと「戻る」でこのページに戻り、ガードが再発動してループする。
      // replace ならこのエントリ自体が履歴から消え、
      // 「戻る」で保護ページに来る前の場所へ素直に抜けられる。
      router.replace(`/login?redirect=${encodeURIComponent(pathname)}`);
    }
  }, [loading, currentUser, pathname, router]);

  if (loading || !currentUser) return null;
  return <>{children}</>;
}
```

ログイン成功側では `redirect` があればそこへ、なければデフォルト（例: `/dashboard`）へ遷移させる。ここも「ログイン画面に戻ってほしくない」強制遷移なので `replace` が適切。

## ポイント

- `router.push(url)` は現在ページを履歴に残して新ページを積む（戻れる）。`router.replace(url)` は履歴上の現在エントリを置き換える（元ページは履歴から消える）。これは Next.js 固有ではなくブラウザの History API（`pushState` / `replaceState`）由来の区別。
- 使い分けの原則は「誰が起こした遷移か」。ユーザーが自分で起こした遷移は `push`（メニュークリック、一覧→詳細）、システムが強制した遷移は `replace`（認証ガード、フォーム送信後の完了画面、使用済みトークンページからの離脱）。
- redirect パラメータは deep link 導線ができた瞬間に必須になる。QR コード経由などで認証必須ページの深い URL を直接開く機能が入ると「元のページへ復帰させる」が要件に昇格する。
- `usePathname()` はクエリ文字列を含まないため、この形で復帰できるのはパス部分のみ。クエリまで復帰させたい場合は `window.location.search` の連結などが必要。

## 参考

（潜在的な小さい UX バグは、それを踏む導線ができた時点で顕在化する——push・redirect なしの初期雛形は、QR 導線の追加で初めて問題化した。YAGNI で後回しにした判断がいつ必要になるかの典型例。）
