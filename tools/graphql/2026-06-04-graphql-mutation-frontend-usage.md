---
title: GraphQL Mutation のフロント側での使い方（一斉送信ダイアログを例に）
date: 2026-06-04
tags: [graphql, mutation, apollo-client, frontend, react, hooks]
---

## 概要

GraphQL Mutation を React + Apollo Client（codegen 生成のフック）で呼び出すフロント実装の解説。初学者向けの「GraphQL フロント使用法サマリー」を冒頭に置き、その後で一斉送信ダイアログを例に、useMutation フックの基本形・Query との違い・loading の意味・送信フロー・補助ロジック・ガード処理までを順に追う。

## 補足：GraphQL フロント使用法のサマリー（初学者向け）

GraphQL をフロントから呼ぶときに最初に押さえておくと読みやすくなるポイントを、一般論として並べておく。

- **GraphQL とは「1 つのエンドポイントに対して、欲しいデータの形を指定して問い合わせる API」**。REST のように URL ごとに用途が分かれていない代わりに、リクエスト本文（クエリ文）で取得・更新内容を表現する。
- **Query と Mutation の役割分担**。
  - Query: データを「取りに行く」操作（GET 相当）。
  - Mutation: データを「変える」操作（POST/PUT/DELETE 相当）。送信・登録・更新・削除は基本的に Mutation。
- **クライアントライブラリの代表が Apollo Client**。React 環境では `useQuery` / `useMutation` などの React Hooks として提供される。
- **codegen（コード生成）**。サーバーのスキーマから、TypeScript の型と専用フック（例: `useFooQuery` / `useBarMutation`）を自動生成するワークフローが一般的。これにより手書きの型ズレや変数名の typo を防げる。
- **Query フックは「マウント時に自動実行」、Mutation フックは「自分で呼ぶまで実行されない」**。同じ「フックを書く」でも挙動が違うのが、最初につまずきやすいポイント。

ここから先は、この前提のうえで「Mutation を実際にどう呼んで使うか」を具体例で見ていく。

## サンプルコード

### 1. mutation フックの基本形

```ts
const [notify, { loading }] = useNotifyEventApplicantsMutation();
```

- 配列の分割代入 + オブジェクトの分割代入の組み合わせ
- フックは `[実行関数, 状態オブジェクト]` を返す
- `notify` … 実行関数（名前は自由に命名できる／位置で受け取るため）
- `{ loading }` … 状態オブジェクトから `loading` だけ取り出し

| 記号 | 種類 | 受け取り方 |
| --- | --- | --- |
| `[ ]` | 配列の分割代入 | 位置（順番） |
| `{ }` | オブジェクトの分割代入 | プロパティ名 |

### 2. 「準備」と「実行」は別物

```ts
const [notify] = useNotify...();    // ① 準備（まだ実行されない）
await notify({ variables: {...} }); // ② 実行（ここで初めて mutation が走る）
```

- フックを書くだけでは実行されない
- `notify(...)` を呼んだ瞬間に実行
- `useState` の `[count, setCount]` → `setCount(5)` と同じ構造

Query との違い:

| 実行タイミング |  |
| --- | --- |
| `useQuery` 系 | フックを書いた瞬間に自動実行 |
| `useMutation` 系 | 手動で呼ぶまで実行されない |

### 3. loading の意味

- `notify` 実行〜レスポンス受領までの送信リクエスト処理中フラグ
- ⚠️ 「メール配信完了まで」ではなく「キューに積み終わるまで」
- 用途：処理中にボタンを disabled にして二重送信・誤操作を防ぐ

### 4. 送信の全体フロー（3層）

```
① フロント: notify(...)              ← 送信を「依頼」
       ↓ GraphQL mutation
② バックエンド mutation              ← Job を「キューに積む」(perform_later)
       ↓ Solid Queue (MySQL)
③ Worker: Job#perform → deliver_now  ← 実際にメール送信 ★
```

| 層 | 場所 | やること |
| --- | --- | --- |
| ① | フロント `notify()` | 送信の依頼（即レスポンス） |
| ② | mutation | キュー投入（enqueue） |
| ③ | Job（Worker） | 実際の配信（`.deliver_now`） |

- フロントのレスポンスは ② キュー投入時点で返る（③ 実送信は待たない）
- → 完了文言は「送信しました」より「順次送信します」が実態に忠実

### 5. 関連する補助ロジック

```ts
// 選択中の宛先を特定（見つからなければ先頭=全員で保険）
const selectedOption = TARGET_OPTIONS.find((o) => o.value === target) ?? TARGET_OPTIONS[0];

// その宛先の人数を取得（countKey で enum値とカウントキーのズレを吸収）
const recipientCount = recipientCounts[selectedOption.countKey];
```

- `?? TARGET_OPTIONS[0]` … `find` の `undefined` を防ぐ型対策＆保険
- `recipientCounts[key]` … ブラケット記法（キーが動的に変わるため）

### 6. 閉じる処理のガード

```ts
const handleClose = () => {
  if (loading) return;  // 送信中は閉じない（背景クリック・ESC対策）
  onClose();
};
```

- ボタンは `disabled={loading}`、閉じる経路は `handleClose` で二重ガード

## ポイント

- Mutation フックは `[実行関数, 状態オブジェクト]` を返す。配列の分割代入で実行関数を、オブジェクトの分割代入で `loading` などを取り出す形。
- フックを書いただけでは何も実行されない。`notify(...)` を呼んだ瞬間に初めて Mutation が走る。これは `useState` の `setCount(5)` と同じ構造。
- Query との決定的な違い: useQuery はフックを書いた瞬間に自動実行、useMutation は手動で呼ぶまで実行されない。
- `loading` は「Mutation 実行〜レスポンス受領まで」のフラグ。**メール配信完了までではなくキューに積み終わるまで**を表す。二重送信防止のための disabled に使う。
- 送信処理は 3層構造: ①フロントが notify で依頼 → ②バックエンド Mutation がキューに積む → ③Worker が `deliver_now` で実配信。フロントは②時点で返るので、UI 文言は「順次送信します」のほうが実態に忠実。
- 補助ロジック: `find(...) ?? デフォルト値` で `undefined` を防ぐ型対策。動的キーでオブジェクトを引くときはブラケット記法（`obj[key]`）。
- 閉じる処理は二重ガード: ボタンは `disabled={loading}`、ダイアログ自体の閉じる経路は `handleClose` で `if (loading) return` を入れて背景クリック・ESC キーでも閉じないようにする。
