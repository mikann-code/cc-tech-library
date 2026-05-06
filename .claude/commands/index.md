---
description: 全記事を走査して README の「最近の記事」セクションを再生成する
---

# /index - 記事インデックスを再生成する

このリポジトリに保存されている全記事を走査し、`README.md` の `<!-- INDEX:START -->` と `<!-- INDEX:END -->` の間を、最新の記事一覧で置き換えてください。

---

## あなた（Claude）が行う処理

### 1. 全記事を収集する

`languages/`, `concepts/`, `tools/`, `daily/` 配下の `.md` ファイルをすべて走査する（`.gitkeep` は除外）。

各ファイルの frontmatter から以下を読み取る。
- `title`
- `date`
- `tags`

### 2. カテゴリ別 + 日付降順でソートする

カテゴリの順序: `languages` → `concepts` → `tools` → `daily`

各カテゴリ内では `date` の **降順**（新しい順）で並べる。

### 3. インデックスを生成する

以下のフォーマットで Markdown を生成する。

\`\`\`markdown
**統計**: 全 N 記事 / 言語 X 記事 / 概念 Y 記事 / ツール Z 記事 / 雑メモ W 記事
最終更新: YYYY-MM-DD

### 🔤 Languages

- \`YYYY-MM-DD\` [<title>](<相対パス>) — \`tag1\` \`tag2\`
- ...

### 💡 Concepts

- ...

### 🛠 Tools

- ...

### 📝 Daily

- ...
\`\`\`

各カテゴリで記事が **0 件なら、そのセクションごと省略** する。
記事が **6 件を超えるカテゴリは、最新 6 件のみ表示** し、最後に `_(他 N 件)_` を追加する。

### 4. README.md を更新する

`README.md` を読み込み、`<!-- INDEX:START -->` と `<!-- INDEX:END -->` の間を上記で生成した内容で置き換える。マーカー外は変更しないこと。

### 5. Git コミット

\`\`\`bash
git add README.md
git commit -m "docs: update article index"
\`\`\`

直前のコミットも README 更新だった場合は \`git commit --amend --no-edit\` でまとめてもよい。

### 6. 完了報告

\`\`\`
✅ インデックスを更新しました
📊 全 N 記事
\`\`\`
