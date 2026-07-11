---
title: TypeScript のジェネレーターと遅延評価で「全部作ってから探す」を「見つけたら止める」に変える
date: 2026-07-10
tags: [typescript, generator, lazy-evaluation, iterator, algorithm]
source: 個人開発
---

## 概要

`function*` + `yield` は関数を途中で一時停止・再開できるようにする仕組み。全組み合わせを列挙して条件に合う最初の1個が見つかった瞬間に打ち切れば、残りの組み合わせは計算すらされない。普通の関数は「呼ばれたら最後まで走り切って結果をまとめて返す」ため生成と消費のタイミングを分離できず、探索が早々に成功しても全列挙の無駄が出る——その無駄をジェネレーターの遅延評価で省ける。

## サンプルコード

まず最小の `yield` の例。関数を呼ぶこと（`range(3)`）と、`next()` で実際に進めることは別物である。

```ts
function* range(n: number): Generator<number> {
  for (let i = 0; i < n; i++) {
    yield i; // ここで一時停止し、値を1つ返す
  }
}

const gen = range(3);   // 関数を呼んだだけ。まだ何も進んでいない
gen.next(); // { value: 0, done: false }  ここで初めて動き出す
gen.next(); // { value: 1, done: false }  前回の続きから再開
```

本題の「全組み合わせ列挙 + 早期 return」。`findFirst` が条件を満たす組を見つけて `return` すると、`combinations` はその時点で停止し、まだ列挙していない組み合わせは一切計算されない。

```ts
function* combinations<T>(arr: T[], k: number): Generator<T[]> {
  if (k === 0) {
    yield [];
    return;
  }
  for (let i = 0; i <= arr.length - k; i++) {
    for (const rest of combinations(arr.slice(i + 1), k - 1)) {
      yield [arr[i], ...rest];
    }
  }
}

function findFirst<T>(items: T[], isOk: (combo: T[]) => boolean): T[] | undefined {
  for (const combo of combinations(items, 3)) {
    if (isOk(combo)) return combo; // 見つけたら即終了。残りの組は生成されない
  }
  return undefined;
}
```

## ポイント

- 一番のうまみは遅延評価（lazy evaluation）。「全部作ってから探す」のではなく「1個作って検査 → ダメなら次 → 見つかったら止める」ができ、早期終了で無駄な計算とメモリ確保を省ける。
- 生成（`combinations`）と消費（`findFirst`）の責務を分離したまま、無駄なく協調させられる。読みやすさと効率を両立できる。
- 同じジェネレーターは `for...of` で1個ずつ消費してもよいし、`[...gen]` で全部まとめて配列化してもよい。呼ぶたびに最初から始まる新しいジェネレーターができる。
- ジェネレーターは無限列も表現できる。`while (true)` の中で `yield` し続けても、呼び出し側が要求した分だけ計算されるので破綻しない。
- 使い分けの勘所: 全要素が必ず必要なら普通の配列で十分。要素数が多く途中で打ち切れる可能性がある探索では遅延評価が効く。

## 参考

- MDN: Generator オブジェクト、`function*` 宣言、iteration protocols
