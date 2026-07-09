---
title: useSubmitGuard — useRef による同期的な二重送信ガード
date: 2026-07-09
tags: [react, apollo-client, use-ref, double-submit, playwright, e2e]
source: 実務(React)
---

## 問題: disabled={loading} だけでは二重送信を防げない

Apollo Client の `useMutation` が返す `loading` は、送信開始の「次 tick」（React の再レンダー後）にならないとボタンの `disabled` に反映されない。そのため高速ダブルクリックでは、1本目の mutation が `loading: true` を反映させる前に2本目のクリックハンドラが実行され、mutation が二重実行される。

**実害の例:**

- 作成系（`createXxx`）→ レコードが2件作成される
- メール送信系（`requestXxx`）→ メールが2通送信される
- トークン発行系 → 2通目がトークンを上書きし、1通目のリンクが無効化される

## 解決: useRef で同 tick 内の2本目を同期的に弾く

`useState` と違い `useRef` は set した瞬間から参照できる（再レンダーを待たない）ので、同 tick 内の2本目を確実に破棄できる。

```tsx
"use client";

import { useCallback, useRef } from "react";

export function useSubmitGuard() {
  const inFlight = useRef(false);

  return useCallback(async <T,>(fn: () => Promise<T>): Promise<T | undefined> => {
    if (inFlight.current) return; // 2本目を確実に破棄
    inFlight.current = true;
    try {
      return await fn();
    } finally {
      inFlight.current = false;
    }
  }, []);
}
```

- `finally` で必ず解除するので、mutation がエラーになっても次の送信はブロックされない
- `useCallback([])` で参照が安定しているため、依存配列に入れても再レンダーを誘発しない
- `disabled={loading}` は**見た目のフィードバックとして併用**する（ガードの代替ではなく補完）

## 使い方

```tsx
const submitGuard = useSubmitGuard();

// onClick ハンドラの場合: 全体を包む
const handleSubmit = () => submitGuard(async () => {
  const result = await createItem({ variables: { input } });
  // ...
});

// form onSubmit の場合: preventDefault はガードの外で行う
const handleSubmit = (event: React.FormEvent<HTMLFormElement>) => {
  event.preventDefault();
  return submitGuard(async () => {
    // ...
  });
};
```

## なぜ useState ではダメか

`useState` の setter は再レンダーをスケジュールするだけで、同 tick 内の次のハンドラ実行時にはまだ古い値が見える（Apollo の `loading` が遅れるのと同じ理由）。同期的なフラグには ref が必須。

## E2E テストの落とし穴（Playwright）

このガードの E2E 検証で得た知見:

1. **`dblclick()` では再現できない。** Playwright の dblclick は2クリックが別タスクで処理され、React 18+ は discrete event の後に同期フラッシュするため、2クリック目の前に `disabled={loading}` が効いてしまう。すり抜けを再現するには **同一タスク内で `element.click()` を2連発**する:

```ts
await locator.evaluate((el) => {
  (el as HTMLElement).click();
  (el as HTMLElement).click();
});
```

2. **`evaluate` は actionability 待機を通らない**ため、ハイドレーション完了前に発火して空振りする（dev モードで顕著）。React がハイドレーション完了時に DOM ノードへ付与する内部キー `__reactProps$` を完了シグナルに使うと確実:

```ts
async function waitForHydration(locator: Locator) {
  await locator.evaluate((el) =>
    new Promise<void>((resolve) => {
      const hydrated = () =>
        Object.keys(el).some((k) => k.startsWith("__reactProps$"));
      if (hydrated()) return resolve();
      const timer = setInterval(() => {
        if (hydrated()) { clearInterval(timer); resolve(); }
      }, 50);
    }),
  );
}
```

3. **検証はネットワーク層で行う。** `page.on("request")` で GraphQL の `operationName` をカウントし、「リクエストが1本しか飛ばない」ことを直接確認する。UI の結果（一覧に1件だけ表示される）だけだと、サーバー側の重複排除と区別できない。

4. **テストが「ガード無しなら落ちる」ことを必ず確認する。** ガードを一時的に外して実行し、リクエストが2本飛んでテストが失敗する（Received: 2）ことを見てから復元する。これを怠ると、そもそもすり抜けを再現できていないトリビアルなテストに気づけない。

## スコープの整理

フロントのガードは同 tick のダブルクリック対策であり、**バックエンドの冪等化の代替にはならない**（別タブからの送信、リトライ、悪意あるクライアントは防げない）。作成系の冪等キーやメール再送スキップはサーバー側で別途対応する。
