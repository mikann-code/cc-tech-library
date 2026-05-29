# 📚 cc-tech-library

> インターンや日々の学習で得た技術知見を、体系的に蓄積していくための個人ナレッジリポジトリ。

このリポジトリは、Claude Code をインターフェースとして「学んだことを Markdown でその場で記録 → 自動分類 → Git でコミット」というフローを最小コストで回すために運用されています。

---

## 🎯 目的

- インターンや日々の学習で得た技術知見を**忘れないうちに記録**する
- 言語・概念・ツール別に**分類された再利用可能な知識ベース**を作る
- Git の履歴を通じて**学習の継続性とプロセス**を可視化する

---

## 📂 ディレクトリ構成

```
.
├── languages/      # 言語別の知見 (typescript, python, go, ...)
├── concepts/       # 言語横断の概念 (DDD, 認証, 設計パターン, ...)
├── tools/          # ツール・インフラ (Docker, Git, CI/CD, ...)
├── daily/          # 日次の雑メモ (後で languages/ や concepts/ に整理)
└── .claude/
    └── commands/   # Claude Code 用カスタムスラッシュコマンド
```

---

## ✍️ 記事フォーマット

各記事は Markdown で記述し、以下の frontmatter を付けます。

```yaml
---
title: 記事タイトル
date: 2026-05-06
tags: [typescript, react, hooks]
source: internship-day-12
---
```

| フィールド | 用途 |
| --- | --- |
| `title` | 記事の見出し |
| `date` | 記録日 (`YYYY-MM-DD`) |
| `tags` | 検索・分類用のタグ |
| `source` | 出典 (インターン何日目、ドキュメント名 など) |

---

## 🚀 使い方 (Claude Code)

このリポジトリを Claude Code で開いた状態で、以下のスラッシュコマンドが使えます。

### `/log`
新しい知見を記録する。Markdown を貼り付けると、Claude が内容を読んで自動で分類・タイトル付け・適切なディレクトリへの保存・Git コミットまで行う。
README には触れない（軽量・高速）。

### `/push`
README の「最近の記事」セクションを最新化したうえで、リモート（GitHub）に push する。
`/log` で貯めた記事をまとめて公開するためのコマンド。差分が無ければ無駄なコミットは作らない。

### `/index` (オプション)
README の「最近の記事」セクションを再生成するだけのコマンド。push はしない。
普段は `/push` で十分なので、手動で README だけ更新したい時に使う。

### 想定ワークフロー

```
昼間: /log → /log → /log    # 学んだ時に記事を貯める
夜:   /push                  # README を最新化して GitHub に公開
```

> コマンドの中身は `.claude/commands/` 配下の `.md` ファイルで定義されています。

---

## 🔒 機密情報の取り扱い

このリポジトリは **公開リポジトリ** として運用しているため、以下の情報は **絶対に書かない**。

- 社内コード、設計ドキュメント、固有名詞、顧客情報
- API キー、認証情報、内部 URL
- インターン先の業務内容のうち、外部公開されていない部分

記録する際は **「学んだ技術的概念」レベルに抽象化** する。
（例: ❌「○○社の API で…」→ ✅「REST API のページネーション設計の考慮点」）

---

## 📰 最近の記事

<!-- INDEX:START -->
全 20 記事 · 最終更新: 2026-05-29

### 🔤 Languages

| 日付 | タイトル |
|------------|------|
| 2026-05-23 | [async/await 完全ガイド - 実務で使う非同期処理のすべて](languages/typescript/2026-05-23-async-await-complete-guide.md) |
| 2026-05-15 | [Ruby のキーワード引数とローカル変数まとめ](languages/ruby/2026-05-15-ruby-keyword-arguments-and-local-variables.md) |
| 2026-05-06 | [TypeScript の Conditional Types の基本](languages/typescript/2026-05-06-sample-typescript-conditional-types.md) |

### 💡 Concepts

| 日付 | タイトル |
|------------|------|
| 2026-05-26 | [encodeURIComponent の必要性](concepts/url-encoding/2026-05-26-encode-uri-component-necessity.md) |
| 2026-05-19 | [パスワード再設定における脆弱性対策（列挙攻撃対策）](concepts/security/2026-05-19-password-reset-user-enumeration-defense.md) |
| 2026-05-18 | [3段階トークンフロー — メール認証で本データを更新する設計パターン](concepts/security/2026-05-18-three-step-token-flow-email-verification.md) |
| 2026-05-18 | [currentUser でデータ取得すべき理由（userId 指定の脆弱性）](concepts/security/2026-05-18-current-user-vs-user-id-idor.md) |

### 🛠 Tools

| 日付 | タイトル |
|------------|------|
| 2026-05-29 | [GraphQL Ruby の Mutation 設計まとめ（BaseMutation / RelayClassicMutation）](tools/graphql/2026-05-29-graphql-ruby-mutation-basics.md) |
| 2026-05-28 | [GraphQL の type について — 学びまとめ](tools/graphql/2026-05-28-graphql-type-overview.md) |
| 2026-05-28 | [graphql-ruby の Enum 型定義の読み解き（EventApplicationStatusType）](tools/graphql/2026-05-28-graphql-ruby-enum-type-definition.md) |
| 2026-05-26 | [Playwright の toHaveURL における正規表現入門](tools/playwright/2026-05-26-playwright-tohaveurl-regex-basics.md) |
| 2026-05-19 | [Devise 慣例（カラム名・テーブル設計の命名規則）](tools/devise/2026-05-19-devise-naming-conventions.md) |
| 2026-05-19 | [Rails のセッションベース認証の仕組み（DB セッションストア）](tools/rails/2026-05-19-rails-session-based-authentication.md) |

_(他 6 件)_

### ❌ Error

| 日付 | タイトル |
|------------|------|
| 2026-05-27 | [E2E テスト失敗原因レポート — Windows cmd.exe + execSync の multi-line コマンドが空振りする](error/e2e-testing/2026-05-27-windows-execsync-multiline-rails-runner-failure.md) |
<!-- INDEX:END -->
