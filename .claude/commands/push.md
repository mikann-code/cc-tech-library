---
description: README の最近の記事セクションを最新化してから git push する
---

# /push - README を更新してリモートへ push する

このコマンドは以下を行います:

1. **bash で全記事の frontmatter のみ抽出**（既存記事の本文は読まない）
2. README の「最近の記事」セクションを再生成
3. 差分があれば README をコミット
4. `git push origin main` で GitHub に反映

**重要: 既存記事の本文を Read ツールで読まないこと。** トークンを大量消費するので、必ず bash + frontmatter 抽出方式を使う。

---

## あなた（Claude）が行う処理

### 1. 事前チェック

- カレントディレクトリがリポジトリのルートか確認（`README.md` と `.claude/` があるか）
- リモート設定の確認: `git remote -v` を実行し、`origin` が設定されているかを確認

  ```bash
  git remote -v
  ```

  `origin` が無ければ、ユーザーに「GitHub にリポジトリを作成して `git remote add origin <URL>` を実行してください」と促して終了。

### 2. README 用インデックス情報を bash で抽出

以下のコマンドで全記事の frontmatter のみを取得する（本文は読まない）:

```bash
for f in $(find languages concepts tools daily -name "*.md" -type f 2>/dev/null | sort); do
  echo "===FILE==="
  echo "$f"
  awk '/^---$/{c++; next} c==1{print} c==2{exit}' "$f"
done
```

このコマンドの出力には、各ファイルのパスと frontmatter（title, date, tags, source）だけが含まれる。

### 3. 出力をパースしてインデックスを生成

#### 3-1. 各記事の情報を抽出

7-1 の出力から、各記事について `path`, `title`, `date`, `tags` を抽出する。

#### 3-2. カテゴリ分類

カテゴリはパスの第1セグメントで判定:
- `languages/...` → languages
- `concepts/...` → concepts
- `tools/...` → tools
- `daily/...` → daily

#### 3-3. ソート

各カテゴリ内で **`date` の降順**（新しい順）に並べる。

#### 3-4. インデックスの Markdown を生成

```markdown
**統計**: 全 N 記事 / 言語 X 記事 / 概念 Y 記事 / ツール Z 記事 / 雑メモ W 記事
最終更新: YYYY-MM-DD

### 🔤 Languages

- `YYYY-MM-DD` [<title>](<相対パス>) — `tag1` `tag2`
- ...

### 💡 Concepts

- ...

### 🛠 Tools

- ...

### 📝 Daily

- ...
```

ルール:
- 0件のカテゴリはセクションごと省略
- 6件を超えるカテゴリは最新6件のみ表示し、最後に `_(他 N 件)_` を追加
- 「最終更新」は今日の日付（`date +%F`）

### 4. README.md を更新

`README.md` を読み込み、`<!-- INDEX:START -->` と `<!-- INDEX:END -->` の間を 3-4 で生成した Markdown で置き換える。マーカー外は絶対に変更しない。

### 5. 差分確認 → README コミット（変更がある場合のみ）

```bash
git diff --quiet README.md || (
  git add README.md && \
  git commit -m "docs: update article index"
)
```

差分が無ければコミットはスキップする（無駄なコミットを作らない）。

### 6. push 前のサマリ表示

push する前に、リモートに送られる予定のコミット一覧をユーザーに見せる:

```bash
git log origin/main..HEAD --oneline 2>/dev/null || git log --oneline
```

ユーザーに「以下のコミットを push します」と表示し、リモート未設定 / 初回 push の場合はその旨も伝える。

### 7. git push

```bash
git push origin main
```

push に失敗した場合（認証エラー、リモート未設定、競合など）はエラー内容をユーザーに分かりやすく伝え、対処方法を提案する。

例:
- 認証エラー → GitHub の認証情報を確認するよう促す
- `non-fast-forward` → `git pull --rebase origin main` を提案
- `no upstream branch` → `git push -u origin main` を提案

### 8. 完了報告

```
✅ push 完了
📤 N 件のコミットをリモートに反映
📊 全 M 記事
🌐 https://github.com/<ユーザー>/<リポジトリ>
```

GitHub の URL は `git remote get-url origin` から取得して https 形式に変換して表示する。

---

## 注意

- **既存記事の本文を Read ツールで読まないこと。** インデックス生成は必ず bash + frontmatter 抽出で行う
- README に差分が無いときは無駄にコミットしない
- push 前に必ずユーザーに送信予定のコミット一覧を見せる（事故防止）
- リモートが未設定の場合は丁寧に案内して終了する
- `<!-- INDEX:START -->` と `<!-- INDEX:END -->` のマーカー外は絶対に変更しない
