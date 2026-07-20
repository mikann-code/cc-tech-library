---
title: Playwright で「2状態のどちらかに落ち着く」ページは locator.or() で待ってから isVisible() で分岐する
date: 2026-07-16
tags: [playwright, e2e, locator, flaky-test]
source: 実務(Next.js)
---

## 概要

読み込み後に「アクションボタンが出る（未完了）」か「完了ラベルが出る（完了）」のどちらかに落ち着くページを E2E テストで分岐したいとき、素朴に `locator.isVisible()` で分岐するとフレークになる。`isVisible()` は auto-waiting しない即時判定なので、ローディング中に呼ぶと両方 DOM に無く false が返り「完了済み」と誤判定するため。先に `expect(a.or(b)).toBeVisible()` で「どちらかが出るまで」待てば、その後の `isVisible()` 分岐が安全になる。

## サンプルコード

`locator.or(other)` で「どちらかにマッチする合成ロケータ」を作り、まずそれが表示されるまで auto-waiting で待つ。状態が確定した後なら `isVisible()` の即時判定を分岐に安全に使える。

```ts
const submitButton = page.getByRole("button", { name: "Submit" });
const completedLabel = page.getByText("Completed");

// ローディングが終わり、どちらかの状態に落ち着くまで待つ
await expect(submitButton.or(completedLabel)).toBeVisible();

// 状態が確定した後なら isVisible() の即時判定で安全に分岐できる
if (await submitButton.isVisible()) {
  await submitButton.click();
  await expect(completedLabel).toBeVisible();
  await page.reload();
}

await expect(completedLabel).toBeVisible();
```

## ポイント

- `expect(...).toBeVisible()` は auto-waiting する。`locator.isVisible()` は即時判定。この非対称を理解しておく。
- 「どちらかの状態に落ち着く」ページの分岐テストは「**or で待つ → isVisible で分岐**」の2段構えが定石。
- この書き方によりテストが前のテストの実行結果（実行順）に依存しなくなり、単体実行（`--grep` 等）でも通る。
- 注意: `.or()` は両方のロケータが同時にマッチすると strict mode violation になり得る。状態が排他的（ボタンと完了ラベルが同時に出ない）なら問題ない。

## 参考

- [Locator.or() - Playwright Docs](https://playwright.dev/docs/api/class-locator#locator-or)
- [Locator.isVisible() - Playwright Docs](https://playwright.dev/docs/api/class-locator#locator-is-visible)
