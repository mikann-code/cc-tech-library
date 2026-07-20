---
title: GitHub の「Resolve conflicts」ボタンはリネームを伴うコンフリクトで無効化される — ローカル git のリネーム検知との差を知る
date: 2026-07-17
tags: [git, github, merge-conflict, rename-detection, merge-tree]
source: 実務(git)
---

## 概要

PR の Web コンフリクトエディタ（「Resolve conflicts」ボタン）が無効化され「Use the command line to resolve conflicts before continuing.」と案内されたら、リネーム/削除/バイナリ/巨大ファイルを含むコンフリクトを疑う。GitHub の Web エディタは「同じパスのファイル同士の単純な内容衝突」しか扱えない。ローカルの git はリネーム検知（rename detection）が効くため、実際に手で直す量は PR 画面の見た目より少ないことが多い。

## サンプルコード

まず、ワーキングツリーを一切汚さずにマージ結果を診断できる `git merge-tree`（git 2.38+）で状況を確認する。出力に自動マージ（Auto-merging）と CONFLICT の別が出るため、「本当に手で直すべきファイル」がマージ前に分かる。

```bash
git fetch
git merge-tree --write-tree origin/<base> HEAD
```

あとは通常どおりローカルでマージして解消する。

```bash
git fetch
git merge origin/<base>
# 内容衝突のファイルだけ手で解消
git commit
git push
```

push すれば PR のコンフリクト表示は消える。

## ポイント

- 「Resolve conflicts」ボタンの無効化は異常ではなく、コンフリクトの**種類**のシグナル。リネーム・削除・バイナリ・巨大ファイルが絡んでいると読み替える（GitHub 公式ドキュメントにも「too complex to resolve on GitHub」との記載がある）。
- GitHub の PR 画面が列挙する衝突ファイル一覧はリネーム検知を考慮しておらず旧パスで列挙されるため、実際の解消コストを過大に見せることがある。慌てる前にローカルで確かめる。
- `git merge-tree --write-tree origin/<base> HEAD` は、ブランチ切り替えや stash なしでマージのドライランができる診断コマンド。大規模なディレクトリ再編（ルーティング構成の移行など）を含むベースブランチへ追従するときに特に有効。

## 参考

- GitHub Docs「Resolving a merge conflict on GitHub」に「too complex to resolve on GitHub」の記載がある（一次情報 URL は未確認）。
