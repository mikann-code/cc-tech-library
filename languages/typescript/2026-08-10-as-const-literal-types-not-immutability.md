---
title: `as const` は「不変にする」ためではなく「リテラル型に固定する」ために付ける
date: 2026-08-10
tags: [typescript, as-const, literal-types, type-narrowing, widening]
source: 個人開発(TypeScript)
---

## 概要

JSX にインラインで書いていたスタイルオブジェクトを、可読性のために変数へ切り出したら型エラーになった。`const` で宣言しているのにプロパティが `string` に広がる（widening）のが原因で、`as const` の目的は「不変にすること」ではなく「リテラル型に固定すること」だと整理した記録。

## サンプルコード

```ts
// インラインなら通る（渡し先の型が文脈から分かるので widening しない）
<Box sx={{ display: "flex" }} />

// 変数に切り出すと通らない
const styles = { display: "flex" };
<Box sx={styles} />
// → string 型は "flex" | "block" | "none" | ... に代入できない

// as const で widening を止める
const styles = { display: "flex" } as const;
// → { readonly display: "flex" }
```

`as const` を付けた定数からは、型を二重管理せずに union 型を取り出せる。

```ts
const items = [
  { value: "draft", label: "下書き" },
  { value: "published", label: "公開" },
] as const;

type Item = (typeof items)[number];          // { readonly value: "draft" | "published"; ... }
type ItemValue = Item["value"];              // "draft" | "published"
```

一部だけ味付けしたい場合は、使う側で spread すれば新しいオブジェクトになるので `readonly` に引っかからない。

```ts
<Box sx={{ ...styles, gap: 2 }} />
```

## ポイント

- `const` 宣言が守るのは**変数の再代入**だけで、プロパティの widening は防げない。`const o = { display: "flex" }` は `{ display: string }` になる（後から書き換えられるため TS が広げる）。変数に切り出して文脈が切れた瞬間に `as const` が必要になる。
- `as const` の目的はリテラル型の獲得であり、`readonly` になるのは手段（副作用）。ランタイムには何も残らず `Object.freeze` でもないため、**不変性の保証としては信頼できない**。
- 「宣言後に新しいプロパティを追加できない」のは `as const` と無関係（型が宣言時に確定するため）。`as const` 固有の効果は「既存プロパティへの代入を禁止する」ことだけ。
- 「不変にしたいから全部に付ける」はアンチパターン。`readonly` は呼び出し側へ伝播するので、`sort()` のような破壊的メソッドが呼べず、`Foo[]`（非 readonly）を要求する関数にも渡せない。外部ライブラリのシグネチャは直せないので詰む。
- 付ける基準は「リテラル型 / union 型として使う定数」「`typeof` で型を導きたい定数」。単に再代入を防ぎたいだけなら `const` 宣言で足り、spread して加工する / state に入れる / 破壊的メソッドを使う場合は付けない。

## 参考

- TypeScript Handbook — `const` assertions（Literal Types の項）
- TypeScript Handbook — Literal Inference / type widening に関する記述
