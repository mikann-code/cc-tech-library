---
title: React の「state 更新 → レンダー → commit → effect」の順序を軸に、マウントとレンダーの違いと useRef / useState の使い分けを導く
date: 2026-08-10
tags: [react, hooks, rendering, useref, usestate]
source: 個人開発(React)
---

## 概要

React（Next.js）アプリを読み解いていて、「この値は state か ref か」「この処理はレンダー中か effect か」の判断がその場の勘になっていることに気づいた。軸は `state 更新 → レンダー（関数を呼ぶ） → DOM 反映（commit） → effect が走る` の 1 本で足り、なぜ effect でしか更新後の state が読めないのか、なぜ「一度だけ実行するガード」は ref でしか書けないのか、なぜ `key` を変えると state が初期化されるのかはすべてここからの帰結として導ける。

## サンプルコード

マウントは**ページ単位ではなく要素単位**。条件付きレンダーの条件が切り替わった瞬間にマウント / アンマウントが起きる。

```jsx
function Parent() {
  const [open, setOpen] = useState(false);

  return (
    <>
      <button onClick={() => setOpen((v) => !v)}>toggle</button>
      {/* false → true の瞬間に Panel はマウントされる */}
      {/* true → false の瞬間にアンマウントされる */}
      {open && <Panel />}
    </>
  );
}
```

React が「同じ要素の再レンダー」か「別物なので作り直し」かを判断する基準は位置と `key`。`key` を変えると強制的にアンマウント → 新規マウントになる。

```jsx
{/* selectedId が変わったら Detail の内部 state をまるごと初期化したい */}
<Detail key={selectedId} id={selectedId} />
```

「一度だけ実行した」のガードは ref でしか正しく書けない。

```jsx
function Item({ id }) {
  const doneRef = useRef(false);

  useEffect(() => {
    if (doneRef.current) return;
    doneRef.current = true; // 代入は同期的に効く

    void send(id);
  }, [id]);

  return null;
}
```

`useRef` は DOM 専用の API ではない。React が特別扱いしているのは `ref` **属性の側**で、`useRef` 自体は何も知らない。

```jsx
function Form() {
  const inputRef = useRef(null); // ref 属性に渡す → React が DOM を入れる
  const timerRef = useRef(null); // 渡さない → 好きな値を入れられる

  return <input ref={inputRef} />;
}
```

## ポイント

- **レンダーの正体はコンポーネント関数の中身が全部再実行されること**。`return` の中だけではなく冒頭の変数計算もハンドラ定義も毎回走る。それでも hooks の値が消えないのは React が関数の外側で保持しているから。この二層構造から、`useState(() => init())` の初期化関数はマウント時だけ実行される（`useState(init())` と書くと毎レンダー計算が走り、結果は捨てられる）ことが出る。また「レンダー」はブラウザのページ再読み込みとは無関係で、単なる関数呼び出しなので state は保持される。
- **レンダーのきっかけは 3 つ**: 自分の state 更新 / 親の再レンダーによる props / **購読している context の値の変化**。3 番目は見落としやすく、自分が何もしていなくても再レンダーする。
- **順序から出る帰結**: effect は commit の後に走るので state が確定している。逆にイベントハンドラ内で `setState` した直後に読んでも古い値のまま（反映は次のレンダー）なので、更新後の値を使う処理は effect に置く。レンダーは何回呼ばれるか保証されず StrictMode では意図的に 2 回呼ばれるため、副作用は effect かイベントハンドラに置く。1 操作中に setter を複数回呼んでもレンダーは 1 回にまとまる（自動バッチ）。
- **マウントとレンダーは別物**。マウント = その要素が初めて現れたときの「初回レンダー + DOM 挿入 + effect の初回実行」で要素ごとに 1 回きり。再レンダーでは state / ref は保持されるが、アンマウント → 再マウントでは初期値に戻り、effect の初回実行がもう一度走る。effect の cleanup はアンマウント時だけでなく、依存が変わって effect が再実行される直前にも走る（StrictMode の二重実行はこの cleanup が正しく書けているかの検出のため）。
- **useRef と useState の判断基準は 1 つ、その値が変わったら画面を描き直す必要があるか**。ref は書き換えても再レンダーせず、代入が同期的に効き、`.current` で読む。だから state のフラグでは（更新が次のレンダーまで反映されず 2 回目が滑り込むため）書けないガードが ref なら成立し、StrictMode の 2 回実行にも有効。ただしガードが効く範囲は「そのコンポーネントインスタンスが生きている間」まで。ref に向くのは送信済みフラグ、`setTimeout` の ID、前回の値、スクロール位置など「画面に出ない値」。

## 参考

- React 公式ドキュメント: Render and Commit
- React 公式ドキュメント: State as a Snapshot
- React 公式ドキュメント: Synchronizing with Effects（cleanup / StrictMode の二重実行）
- React 公式ドキュメント: Preserving and Resetting State（位置と `key`）
- React 公式ドキュメント: Referencing Values with Refs / Manipulating the DOM with Refs
