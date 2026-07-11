---
title: git merge で出る「CONFLICT (file location)」は置き場所の競合、git add だけで解決できる
date: 2026-07-10
tags: [git, merge, conflict, rename, refactoring]
source: 実務(git)
---

## 概要

ディレクトリを丸ごと移動したブランチを `git merge` で取り込むと、旧ディレクトリに足していた新規ファイルが「移動先に置くべきでは？」と検知され、`CONFLICT (file location)` という競合になる。これは `both modified` のような中身の競合ではなく「置き場所の競合」で、移動先パスを `git add` するだけで解決できる。

## サンプルコード

`develop` 側で `tests/` を `tests-v2/` へ移動、自分のブランチでは旧 `tests/` に新規ファイルを追加していた状態で `git merge develop` すると、次のような競合が出る。

```
CONFLICT (file location): tests/new-feature.spec.ts added in HEAD inside a directory
that was renamed in develop, suggesting it should perhaps be moved to tests-v2/new-feature.spec.ts.
```

git は新規ファイルを移動先 `tests-v2/` に配置し、`Unmerged paths` に `added by us` としてマークして停止する。移動先の配置を受け入れるなら、移動先パスを `git add` するだけで解決マークが付く。

```bash
git add tests-v2/new-feature.spec.ts
```

別の場所に置きたい場合は `git mv` で移してから add する。

```bash
git mv tests-v2/new-feature.spec.ts some/other/dir/new-feature.spec.ts
git add some/other/dir/new-feature.spec.ts
```

## ポイント

- git 2.18 以降の「directory rename detection（ディレクトリ改名検知）」が原因。片方が `A/` を `B/` へ移動し、もう片方が `A/` 配下に新規ファイルを足すと、git は「その新規ファイルも `B/` に置くべきでは？」と推測する。
- `CONFLICT (file location)` は中身の競合ではなく置き場所の競合。ファイルの中身は自分のブランチのままで、パスだけが移動先候補に置かれている。
- `git add` は「この内容・この場所で解決済みと宣言する」意味であって、中身が新構造に合っているかは別問題。新しい設定・ヘルパーを前提にした構造なら、中身の書き直しは競合解決とは別のコミットにするとレビューしやすい。
- 改名検知は万能ではなく、rename の閾値や追加ファイル数によっては検知されないこともある。その場合は自分で `git mv` する。

## 参考

- `git help merge` の "DIRECTORY RENAME DETECTION" 節（directory rename detection は git 2.18 で導入）
