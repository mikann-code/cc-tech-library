---
title: git checkout と git pull は別物という話
date: 2026-05-07
tags: [git, basics, internship]
source: インターン中の PR マージ作業での気づき
---

## 何でつまずいたか

PR をマージした後、developの最新を取得する手順で以下のコマンドを順に叩いた。

```bash
git checkout develop
git pull origin develop
```

このとき「これって同じ動作じゃないの？」と疑問に思った。
両方ともdevelopに関する操作に見えたから。

## 実は全然違う動作だった

| コマンド | 何をするか |
| --- | --- |
| `git checkout develop` | 今いるブランチを develop に切り替える |
| `git pull origin develop` | リモートの develop の最新を、ローカルに取り込む |

「移動する」のと「中身を更新する」は別の動作。

## Git には3つの世界がある

混乱したときはこの図を思い出す。

```
[リモート（GitHub）]      ← 本物の最新
       ↑↓
[ローカルのリモート追跡]   ← 自分のPCにあるリモートのコピー（origin/develop）
       ↑↓
[ローカルのブランチ]       ← 自分が作業する場所（develop）
```

- `git checkout` は「ローカルのブランチ」内での移動
- `git pull` は「リモート → ローカル」へのデータ取り込み

別レイヤーの動作なので、両方必要。

## たとえ話

- `git checkout develop` = 書斎に移動する
- `git pull origin develop` = 郵便受けから新しい手紙を取ってくる

書斎に入っただけでは、新しい手紙は読めない。
入った後で郵便受けを確認しないと、最新情報は手に入らない。

## なぜ pull だけでは足りないか

`git pull` は「**今いるブランチに対して**」動作する。

たとえば `feature/issue-21` にいるまま `git pull origin develop` を実行すると、
develop の変更が `feature/issue-21` にマージされてしまう。

だから「develop を最新にしたい」なら、先に develop に移動する必要がある。

## pull の中身

ちなみに `git pull` は内部的に2つのコマンドを実行している。

```bash
git pull origin develop
# ↓ 実は内部でこう動いている
git fetch origin develop   # リモートから情報を取得
git merge origin/develop   # 取得した情報を今のブランチにマージ
```

`fetch` がリモート → ローカル追跡への取り込み、
`merge` がローカル追跡 → ローカルブランチへの統合。
これも「3つの世界」の図を思い出すと納得できる。

## 学んだこと

- 似たようなコマンドでも、裏で何が起きてるかは別物
- 「3つの世界（リモート / ローカル追跡 / ローカルブランチ）」を意識すると Git のエラーが理解しやすい
- 「同じじゃない？」と疑問に思ったときは、裏側の動作を調べると深く理解できる
