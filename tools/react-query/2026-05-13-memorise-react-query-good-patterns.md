---
title: MemoRise の React Query うまく使えている部分まとめ
date: 2026-05-13
tags: [react-query, react, typescript, cache, hooks]
source: MemoRise プロジェクトのコードレビュー
---

## 概要

MemoRise プロジェクトで React Query をうまく使えている部分のまとめ。Provider 配置、`queryKey` 設計、`enabled` ガード、Mutation 後のキャッシュ無効化、`setQueryData` / `removeQueries` の活用、カスタムフックでの関心分離、型付け、ロード／エラー状態の扱い、権限ごとのキー分離まで、王道パターンを全体で踏襲できている。

## サンプルコード

`useWordbooks` の `updateWordbookMutation` で、一覧と詳細の両方を同時に無効化している例。

```ts
// useWordbooks の updateWordbookMutation
queryClient.invalidateQueries({ queryKey: ["wordbooks"] });
queryClient.invalidateQueries({ queryKey: ["wordbook", variables.uuid] });
```

## ポイント

### 1. Provider をアプリ全体に正しく適用できている

- `src/providers/ReactQueryProvider.tsx` で `QueryClient` を生成し、`QueryClientProvider` で包んでいる
- `RootLayout`（`src/app/layout.tsx`）でアプリ全体をラップしているので、どのコンポーネントからでも `useQuery` / `useMutation` が動く
- `"use client"` 指定もちゃんと付いている（Server Component で React Query を使おうとして壊れる、というよくあるミスを回避できている）
- デフォルトで `retry: false` / `refetchOnWindowFocus: false` を入れて、過剰な再フェッチを抑えている

### 2. `queryKey` の設計が一貫している

- 取得対象の名前 + パラメータ、の組で構成されている
  - 例：`["wordbook", uuid]`, `["words", wordbookUuid]`, `["publicWordbookChildren", parentUuid]`
- パラメータが変わると別キャッシュとして扱われるため、別の単語帳を開いたときに前のデータが混ざらない
- 単数形（詳細）と複数形（一覧）で `["wordbook", uuid]` / `["wordbooks"]` のように使い分けている

### 3. `enabled` でムダなリクエストを止めている

- `useWords`, `usePublicWords`, `usePublicWordbookChildren`, `usePublicWordbook`, `useAdminWords` などで `enabled: !!uuid` を指定
- パラメータが揃っていない初回レンダリングで 400 系エラーが飛ぶのを防げる
- `usePublicWords` では `enabled` に加えて `queryFn` 側でも空配列を返すフォールバックを入れていて二重ガード

### 4. Mutation 成功時のキャッシュ無効化ができている

- 作成・更新・削除のあとに `queryClient.invalidateQueries({ queryKey: [...] })` を呼んで一覧を再取得している
- 一覧と詳細の両方を同時に無効化している箇所もある
- `useAdminWordbooks` で子作成成功時に親一覧と子一覧の両方を更新するなど、関連キャッシュを意識できている

### 5. `setQueryData` でキャッシュを直接更新している

- `useUpdateProfile` で `qc.setQueryData(["me"], updatedUser)` を呼んでいる
- これによりプロフィール更新後の再フェッチが不要になり、画面のチラつきが減る
- 楽観的更新の入口として良い形（型を `User` に揃えればより安全）

### 6. ログアウト時にキャッシュをクリアしている

- `useLogout` で `queryClient.removeQueries({ queryKey: ["me"] })` を実行
- ログアウト後に「前のユーザーの自分情報」がチラ見えする事故を防いでいる
- React Query をただ使うだけでなく、認証状態と結びつけて運用できている

### 7. カスタムフックで関心の分離ができている

- API 通信ロジックは `src/lib/*.ts`、それを呼ぶ Query 設定は `src/hooks/use*.tsx` に分離
- コンポーネントは `useMe()`, `useWordbooks()` のように 1 行で必要なデータを取れる
- 同じデータを別画面で使い回しても、React Query のキャッシュが自動で効くので無駄なリクエストが出ない

### 8. TypeScript ジェネリクスで型安全を担保している

- `useQuery<Wordbook[]>`, `useMutation<LoginResponse, Error, LoginParams>` のように型引数を明示
- レスポンス・エラー・引数すべての型が IDE で補完される
- `data: words = []` のデフォルト値で `undefined` を排除し、コンポーネント側のコードがスッキリ

### 9. ローディング・エラー状態を統一的に扱える

- `isLoading`, `isError` を返り値に含めている
- Mutation 側でも `isPending` を `creatingParent` などにマップして UI に渡している
- ボタンの二重押下防止やスピナー表示が簡単に書ける

### 10. 役割ごとに Query を分けている

- 一般ユーザー用（`useMe`）と管理者用（`useAdminMe`）で `queryKey` を別にしている
- 公開系（`usePublicWordbooks`）と認証系（`useWordbooks`）も別キー
- 権限境界がキャッシュ単位でも明確になっており、片方をログアウトしてももう片方に影響しない

### 総評

- 「`useQuery` で取って `useMutation` で更新、終わったら `invalidateQueries`」という王道パターンを全体で踏襲できている
- `enabled` / `setQueryData` / `removeQueries` といった一歩進んだ使い方にも手を出せている
- カスタムフック化と型付けが丁寧なので、画面側のコードが宣言的で読みやすい
