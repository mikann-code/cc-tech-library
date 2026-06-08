---
title: React の event と、選択肢ハンドラで e を使わない理由
date: 2026-06-08
tags: [react, event, typescript, mui, event-handler]
---

## 概要

React の event オブジェクトとは何か、どんなときに使い、どんなときに使わないかの整理。MUI の ToggleButtonGroup のように「ラベル付き選択肢を選ぶ UI」では、event ではなく第2引数の `value` を使うため、第1引数を `_` で受け流す書き方になる。

## event とは何か

クリックや入力などの「操作」が起きたときに、ブラウザが自動で作って渡してくる詳細情報の箱。

```tsx
onClick={(event) => { ... }}
//        ↑ ブラウザが自動で渡す。「どの要素を・どこで・どう操作したか」が入っている
```

中身の例:

| プロパティ / メソッド | 中身 |
| --- | --- |
| event.target / event.currentTarget | 操作された要素 |
| event.target.value | input に入力された値 |
| event.key | 押されたキー（"Enter" など） |
| event.clientX / clientY | クリック座標 |
| event.preventDefault() | ブラウザのデフォルト動作を止める |
| event.stopPropagation() | イベントの伝播（親への伝わり）を止める |

## event を「使う」ケース

event の中身が必要なときだけ受け取って使う。代表的な例:

| 場面 | 使うもの | 例 |
| --- | --- | --- |
| 入力値を取得 | event.target.value | 検索ボックス・名前入力 |
| 押されたキーを判定 | event.key | Enter で確定 |
| デフォルト動作を止める | event.preventDefault() | フォーム送信でリロードさせない |
| 入れ子クリックの制御 | event.stopPropagation() | 行クリックの中のボタン |
| 表示位置の取得 | event.currentTarget | ボタン基準のメニュー表示 |

## event を「使わない」ケース（今回）

```tsx
const handleFilterChange = (_: React.MouseEvent<HTMLElement>, value: RoleFilter | null) => {
  setRoleFilter(value ?? "");
  setPage(1);
};
```

今回は「ラベル付きの選択肢を選ぶ UI」だから。ユーザーは文字を入力するのではなく、あらかじめ決められた選択肢を押すだけ。

```tsx
<ToggleButton value="organizer">主催者</ToggleButton>
//             ↑ 値             ↑ 表示ラベル
```

| 表示ラベル | value（受け取る値） |
| --- | --- |
| 主催者 | "organizer" |
| 参加者 | "participant" |
| 管理者 | "admin" |
| すべて | "" |

押された選択肢の value は MUI が第2引数で直接渡してくれる。フィルタに必要なのは「どのロールが選ばれたか」＝ value だけなので、event（クリックの詳細）を読む必要がない。

## 「入力値」と「選択肢の値」の違い

|  | 入力値（input） | 選択肢の値（今回） |
| --- | --- | --- |
| 値の出どころ | ユーザーが自由に打ち込む | 決められた選択肢から選ぶ |
| 取り方 | event.target.value（event 必要） | onChange の第2引数 value（event 不要） |
| 値の種類 | 任意の文字列 | 決まった数種類だけ |
| 例 | 検索ボックス | フィルタタブ・ラジオボタン |

自由入力なら event が要るが、選択肢を選ぶだけなら value で完結する。

## `_`（アンダースコア）の意味

```tsx
(_: React.MouseEvent<HTMLElement>, value: RoleFilter | null) => { ... }
//  ↑ 使わない第1引数
```

- 引数の順番は呼ぶ側（MUI）が決めている → `(event, value)` の順で渡ってくる
- 2番目の `value` を受け取るには、1番目の枠を飛ばせない
- 使わない第1引数（event）を `_` という名前にして「受け取るが使わない」と明示する慣習

## ポイント

- event は「操作の詳細情報の箱」で、入力値・キー・伝播制御などが必要なときに使う
- ラベル付き選択肢を選ぶ UI（MUI ToggleButtonGroup など）では、必要なのは `value`（選ばれた値）だけなので event は使わない
- MUI の `onChange` は `(event, value)` の順で渡してくるので、2番目を受けるには1番目を飛ばせない
- 使わない第1引数は `_` という名前で受け流すのが慣習
