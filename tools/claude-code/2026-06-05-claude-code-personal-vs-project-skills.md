---
title: Claude Code の個人スキルとプロジェクトスキルの違いと活用方法
date: 2026-06-05
tags: [claude-code, skills, customization, workflow, team-collaboration]
source: 公式ドキュメント https://code.claude.com/docs/en/skills.md
---

## 概要

Claude Code の「Skill」機能は、よく使う手順や指示をスラッシュコマンド的に呼び出せる仕組み。保存場所によって **個人スキル** と **プロジェクトスキル** に分かれ、適用範囲・共有範囲・Git 管理の仕方が変わる。本記事はその違いと、どちらに何を置けばうまく回るかの指針をまとめる。

## サンプルコード

### ディレクトリ構造

スキルは保存場所で 4 レベルに分かれる。

| レベル | パス | 適用範囲 |
| --- | --- | --- |
| 企業（Enterprise） | 組織の managed settings | 全ユーザー |
| **個人（Personal）** | `~/.claude/skills/<skill-name>/SKILL.md` | 自分の全プロジェクト |
| **プロジェクト（Project）** | `<repo>/.claude/skills/<skill-name>/SKILL.md` | そのリポジトリのみ |
| プラグイン | `<plugin>/skills/<skill-name>/SKILL.md` | プラグイン有効時 |

同名スキルがあった場合の優先順位は **企業 > 個人 > プロジェクト**。

### SKILL.md の基本フォーマット

各スキルは独立したディレクトリに置かれ、`SKILL.md` が必須エントリポイントになる。

```markdown
---
name: my-skill                    # 表示名（省略可、デフォルトはディレクトリ名）
description: スキルの説明          # Claude の自動呼び出しトリガー判定用
disable-model-invocation: true    # Claude の自動呼び出しを禁止（手動 /xxx 限定にする）
allowed-tools: Bash Read          # このスキル実行中に許可するツール
---

## 実装内容

実際の指示内容を Markdown で書く。
ここに「やってほしい手順」「ルール」「コードテンプレート」などを並べる。
```

### 個人スキルに向く例

```
~/.claude/skills/
├── log/SKILL.md              # 学習メモの保存ワークフロー
├── push/SKILL.md             # README 更新 + git push
├── commit-template/SKILL.md  # 自分の好きなコミット形式
└── pr-review/SKILL.md        # 自分用の PR レビュー観点リスト
```

→ どのプロジェクトでも同じやり方で使いたい、自分専用の手順を集める場所。

### プロジェクトスキルに向く例

```
<repo>/.claude/skills/
├── deploy/SKILL.md           # このプロダクト固有の本番デプロイ手順
├── seed/SKILL.md             # このリポジトリの seed データ投入手順
├── domain-glossary/SKILL.md  # ドメイン用語集
└── runbook/SKILL.md          # 障害対応 runbook
```

→ チーム全員で同じ操作・同じ判断ができるよう、リポジトリに commit して共有する。

## ポイント

- **個人スキルとプロジェクトスキルの最大の違いは「どこに置くか」と「誰と共有するか」**。個人は `~/.claude/skills/`（自分専用・全プロジェクト横断）、プロジェクトは `<repo>/.claude/skills/`（そのリポジトリ限定・チーム共有）。
- **個人スキルに向くもの**: 複数プロジェクトで使い回したい汎用手順（学習メモの保存、コミット定型、PR レビュー観点、push の前処理など）。プロジェクトに依存しない自分の作業スタイル。
- **プロジェクトスキルに向くもの**: そのリポジトリ固有の知識・手順（デプロイ、seed 投入、ドメイン用語、障害対応 runbook）。チームで「同じやり方を強制したい」もの。Git に commit してレビューを通すことで、属人化を防げる。
- **優先順位は「企業 > 個人 > プロジェクト」**。同名スキルがあると、より上位のものが優先される。プロジェクト側で個人スキルの挙動を上書きしたい場合は、別名にして両方持つ運用が無難。
- **SKILL.md は YAML frontmatter + 本文の Markdown**。`description` フィールドが Claude が「このスキルを呼ぶべきか」を判断するキーになる。曖昧な description だと自動呼び出しが暴発・取りこぼしを起こすので、起動条件を具体的に書く。
- **`disable-model-invocation: true`** にすると Claude の自動判断では呼ばれず、ユーザーが明示的に `/skill-name` と打ったときだけ起動する。事故防止が必要なスキル（破壊的操作、リソース消費が大きい処理）には付けておくと安心。
- **`allowed-tools`** で実行時のツールを絞れる。読み取り専用スキルなら `Read Grep Glob` だけに絞るなど、最小権限の原則を skills レベルで効かせられる。
- **チーム共有時の注意**: プロジェクトスキルを commit したリポジトリで `claude` を起動すると、初回に workspace trust の確認が出る。`allowed-tools` の設定はその信頼境界に直結するので、PR レビュー時にスキル本体だけでなく frontmatter まで確認するフローを作るとよい。
- **使い分けの実践指針**: まず個人スキルで「自分が繰り返している作業」を抽出 → そのうちチームでも標準化したいものをプロジェクトスキルに昇格、という流れが運用しやすい。最初からプロジェクト側に置こうとすると、まだ手順が固まっていないスキルを公開してしまいやすい。
