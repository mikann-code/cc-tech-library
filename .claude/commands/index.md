---
description: README の「最近の記事」セクションを再生成する（push なし）
---

# /index - 記事インデックスを再生成する

`/push` のうち、README 更新部分だけを行うコマンド。push はしない。
普段は `/push` を使えば良いので、このコマンドは「push せずに README だけ手動で更新したい」時用。

**重要: 既存記事の本文を Read ツールで読まないこと。** 必ず bash + frontmatter 抽出方式を使う。

---

## あなた（Claude）が行う処理

### 1. bash で全記事の frontmatter のみ抽出

```bash
for f in $(find languages concepts tools daily -name "*.md" -type f 2>/dev/null | sort); do
  echo "===FILE==="
  echo "$f"
  awk '/^---$/{c++; next} c==1{print} c==2{exit}' "$f"
done
```

### 2. 出力をパースしてインデックスを生成（テーブル形式）

#### 2-1. 各記事の情報を抽出

bash 出力から、各記事について `path`, `title`, `date` を抽出する。

#### 2-2. カテゴリ分類

パスの第1セグメントで判定:
- `languages/...` → languages
- `concepts/...` → concepts
- `tools/...` → tools
- `daily/...` → daily

#### 2-3. ソート

各カテゴリ内で **`date` の降順**（新しい順）に並べる。

#### 2-4. Markdown を生成

```markdown
全 N 記事 · 最終更新: YYYY-MM-DD

### 🔤 Languages

| 日付 | タイトル |
|------------|------|
| YYYY-MM-DD | [<title>](<相対パス>) |
| ... | ... |

### 💡 Concepts

| 日付 | タイトル |
|------------|------|
| ... | ... |

### 🛠 Tools

| 日付 | タイトル |
|------------|------|
| ... | ... |

### 📝 Daily

| 日付 | タイトル |
|------------|------|
| ... | ... |
```

ルール:
- 0件のカテゴリはセクションごと省略
- 6件を超えるカテゴリは最新6件のみテーブルに表示し、テーブルの**下に** `_(他 N 件)_` を追加
- 「最終更新」は今日の日付（`date +%F`）
- 各カテゴリの絵文字: Languages = 🔤、Concepts = 💡、Tools = 🛠、Daily = 📝
- テーブルのタイトルセル内にパイプ `|` が含まれる場合は `\|` にエスケープ

### 3. README.md を更新

`<!-- INDEX:START -->` と `<!-- INDEX:END -->` の間を 2-4 で生成した内容で置き換える。マーカー外は変更しない。

### 4. 差分があればコミット

```bash
git diff --quiet README.md || (
  git add README.md && \
  git commit -m "docs: update article index"
)
```

push はしない。push したい時は `/push` を使うか、手動で `git push origin main`。

### 5. 完了報告

```
✅ インデックスを更新しました
📊 全 N 記事
ℹ  push は /push で行ってください
```
