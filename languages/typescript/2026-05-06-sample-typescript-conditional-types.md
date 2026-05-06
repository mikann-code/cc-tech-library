---
title: TypeScript の Conditional Types の基本
date: 2026-05-06
tags: [typescript, types, conditional-types]
source: sample-entry
---

> このファイルは記事の書き方サンプルです。`/log` コマンドで実際に記録を始めるときの参考にしてください。

## 概要

Conditional Types は TypeScript の型レベルの三項演算子。

```ts
type IsString<T> = T extends string ? true : false;

type A = IsString<"hello">; // true
type B = IsString<42>;      // false
```

## どこで使うか

- ジェネリック関数の戻り値型を入力に応じて分岐させたいとき
- `infer` と組み合わせて、関数の戻り値型や Promise の中身を取り出すとき

```ts
type ReturnTypeOf<T> = T extends (...args: any[]) => infer R ? R : never;

type Fn = () => number;
type R = ReturnTypeOf<Fn>; // number
```

## 注意点

- 条件型は **distributive**（分配的）で、ユニオン型に対しては各要素ごとに評価される
- 分配を抑止したいときはタプルでラップする: `[T] extends [U] ? ... : ...`

## 参考

- TypeScript Handbook: Conditional Types
