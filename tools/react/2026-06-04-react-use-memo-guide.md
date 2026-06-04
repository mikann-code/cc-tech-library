---
title: useMemo の使い方と注意点（React Hooks）
date: 2026-06-04
tags: [react, useMemo, hooks, memoization, performance]
---

## 概要

useMemo は計算結果をキャッシュ（メモ化）する React Hook。依存配列の値が変わらない限り、前回の計算結果を再利用し、関数を再実行しない。主な用途は「重い計算を避ける」ことと、オブジェクト・配列・関数の「参照同一性（referential identity）を保つ」ことの2つ。

## サンプルコード

### 基本構文

```jsx
const result = useMemo(() => 重い計算(a, b), [a, b]);
```

- 第1引数: 値を返す関数（ファクトリ関数）
- 第2引数: 依存配列（dependency array）

a か b が変わったときだけ 重い計算 を再実行し、変わらなければ前回の戻り値をそのまま返す。

### React の再レンダー時の挙動

React のコンポーネントは、state や props が変わるたびに関数全体が頭から再実行される。つまりコンポーネント内の const も毎回再評価される。

```jsx
function Component() {
  const x = heavyCalc();  // ← レンダーのたびに毎回走る
  return <div>{x}</div>;
}
```

毎レンダー走っても問題ない軽い処理なら不要だが、以下のケースで useMemo が効く。

### 用途A: 重い計算を避ける

```jsx
const sorted = useMemo(() => hugeArray.sort(compareFn), [hugeArray]);
```

hugeArray が変わらなければソートをスキップ。

### 用途B: 参照の同一性を保つ

JavaScript ではオブジェクト・配列・関数は参照比較される。

```js
[] === []        // false（中身は同じでも別物）
{} === {}        // false
```

毎レンダーで新しい配列/オブジェクトを作ると、それを使う側（useMemo・useEffect の依存配列、React.memo した子コンポーネントの props）が「変わった」と誤認して再計算・再レンダーする。

```jsx
const participants = useMemo(() => data?.organizerEventApplications ?? [], [data]);
```

これは用途B。`?? []` が毎回新しい `[]` を作るのを防ぎ、後続の useMemo を正しく機能させるのが目的。

### 依存配列の3パターン

```jsx
useMemo(() => f(), [a, b]);  // a か b が変わったら再計算
useMemo(() => f(), []);      // 初回だけ。以降ずっと同じ値
useMemo(() => f());          // 依存配列なし → 毎回再計算（メモ化の意味なし=使う意味なし）
```

依存配列の比較は `Object.is`（≒ `===`）による浅い比較。だから「参照の同一性」が重要になる。

### 二段構えのメモ化（実例）

```jsx
// data が変わったときだけ新しい配列 → 参照が安定する
const participants = useMemo(() => data?.organizerEventApplications ?? [], [data]);

// participants が安定しているので、検索/フィルタ条件が変わったときだけ再計算される
const filtered = useMemo(() => {
  return participants.filter(...);
}, [participants, search, statusFilter]);
```

participants の useMemo が土台（安定した参照）を作り、その上で filtered の useMemo が成立する、という二段構え。

## ポイント

- useMemo は「依存配列が変わらない限り計算結果を再利用する」Hook。第1引数が値を返すファクトリ関数、第2引数が依存配列。
- 主な用途は2つ:「重い計算を避ける」と「参照の同一性（referential identity）を保つ」。後者は JavaScript のオブジェクト/配列/関数が参照比較されるため重要。
- 依存配列の比較は `Object.is` による浅い比較。毎レンダーで新しい配列/オブジェクトを作ると、それを使う側が「変わった」と誤認する。
- とりあえず全部 useMemo で包むのは NG。useMemo 自体にコスト（関数生成・配列比較・メモリ保持）があり、プリミティブや軽い計算ではむしろ遅くなる。
- ファクトリ関数は純粋に値を計算して返すだけにする。副作用（API 通信、DOM 操作）は useEffect の仕事。
- React はメモリ削減のためキャッシュを破棄することがあると公式が明言。useMemo はあくまで最適化で、「必ずキャッシュされる」前提のロジックは組まない。
- 関数内で使う値は全部依存配列に入れるのが原則。漏れると古い値（stale value）を参照するバグになる。`eslint-plugin-react-hooks` の `exhaustive-deps` で検出可能。
- 似た Hook: `useCallback(fn, deps)` は関数そのものをキャッシュ（`useMemo(() => fn, deps)` と等価）。`useRef(initial)` は任意の可変値を再レンダーをまたいで保持（変更しても再レンダーしない）。
