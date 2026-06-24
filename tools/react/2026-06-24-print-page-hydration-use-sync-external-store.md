---
title: 印刷ページの実装まとめ（@media print / Next.js ハイドレーション / useSyncExternalStore）
date: 2026-06-24
tags: [react, nextjs, hydration, use-sync-external-store, print-css, mui]
---

## 概要

主催者がイベントの確定参加者を A4・1 枚の「受付シート」として印刷するためのページ（`participants/page.tsx`）の実装まとめ。画面では操作ボタン付きで表示し、印刷時はボタンやサイドバーを消して紙に最適化する。印刷専用 CSS、Next.js のハイドレーションエラー回避、`useSyncExternalStore` の「裏技的」使い方の 3 点が要点。

## サンプルコード

### printStyles（MUI の GlobalStyles で印刷専用スタイルを当てる）

印刷専用のスタイルは `printStyles` にまとめ、このページが表示されている間だけ有効にする（他ページには影響しない）。

| 要件 | やっていること | コード |
| --- | --- | --- |
| 用紙サイズ A4 | 用紙と余白を指定 | `@page { size: A4; margin: 12mm }` |
| 印刷時に UI を隠す | サイドバー・ヘッダー・`.no-print` を非表示 | `display: none !important` |
| ヘッダー背景色を維持 | 印刷時も色を残す | `print-color-adjust: exact` |
| ページ溢れ防止 | 紙では高さを自動に | `.print-sheet { min-height: auto !important }` |

### 出力日のハイドレーション問題を `useSyncExternalStore` で回避

```tsx
const mounted = useSyncExternalStore(
  () => () => {}, // 購読しない（空の解除関数を返すだけ）
  () => true,     // クライアント → true
  () => false,    // サーバー/ハイドレーション中 → false
);
const printedAt = useMemo(() => (mounted ? new Date() : null), [mounted]);
```

`mounted` が `false` の間は時刻を出さず（`null`）、ブラウザでマウント完了して `true` になってから `new Date()` を計算する。これでサーバーとブラウザの食い違いが起きない。

### `useSyncExternalStore` のシグネチャ

```ts
useSyncExternalStore(
  subscribe,          // ① 変化を購読し、解除関数を返す
  getSnapshot,        // ② 現在の値（クライアント）
  getServerSnapshot?, // ③ サーバー用の値（SSR）
);
```

### 本来の使い方（例：オンライン状態）

```ts
useSyncExternalStore(
  (cb) => {
    window.addEventListener("online", cb);
    window.addEventListener("offline", cb);
    return () => { /* 解除 */ };
  },
  () => navigator.onLine, // 今の値
  () => true,
);
```

## ポイント

- **`@media print { ... }`** の中のスタイルは印刷（プレビュー含む）のときだけ適用される。画面表示には影響しないので、「画面では 1 ページ風に見せる」「紙では余白を詰める」という出し分けができる。
- **className との対応**：`.no-print` は消したい要素（操作バー）に付け、`.print-sheet` は印刷対象の本体（Paper）に付ける。CSS 側の `.no-print` / `.print-sheet` と JSX 側の className がペアになって効いている。
- **ハイドレーション問題の原因**：Next.js は ①サーバーで HTML を作り → ②ブラウザでそのHTMLに React を結合（ハイドレーション）する。`new Date()` は ① と ② で時刻が微妙にズレるため、サーバーが作った HTML とブラウザが期待する HTML が食い違いエラーになる。
- **`useSyncExternalStore` は React 18+ の公式組み込みフック**。本来は「React の外で勝手に変わる値（ブラウザ API や外部ストア）」を安全に取り込むためのもの。
- **今回の「裏技的」な使い方**：購読すべき外部の変化が無く、「サーバーなら `false` / クライアントなら `true`」という ②③ の差だけを利用して「マウント済みか」を判定している。だから ① は空（`() => () => {}`）になっている。
- **イメージ**：`useSyncExternalStore` は React が用意した「3 口の差し込み口がある機械」。そこにどんな部品（関数）を挿すかは自分で決める。今回は「マウント判定用の自作部品」を挿している。
