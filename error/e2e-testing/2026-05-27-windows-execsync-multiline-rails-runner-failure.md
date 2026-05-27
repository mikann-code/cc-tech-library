---
title: E2E テスト失敗原因レポート — Windows cmd.exe + execSync の multi-line コマンドが空振りする
date: 2026-05-27
tags: [error, e2e, playwright, node, windows, execsync, rails-runner]
---

## 対象

- 失敗ファイル: `packages/frontend/e2e/dashboard-applications.spec.ts`
- 失敗件数: 7 / 8 (Empty State の 1 件だけパス)
- 並走させた `e2e/public-event-apply.spec.ts` は 6/6 パス
- 実行環境: Windows 11 + PowerShell + Node `child_process.execSync`

## 症状

7 件すべて、ダッシュボードに E2E公開イベントカード (あるいはそこから派生する「詳細を見る」「キャンセル」「Googleマップで開く」要素) が **表示されない** ため落ちる。

```
Locator: getByText('E2E公開イベント')
Expected: visible
Error: element(s) not found
```

ページスナップショットでは常に Empty State (現在、申し込み済みのイベントはありません) がレンダリングされている。

## 根本原因

> Windows の cmd.exe が `execSync` の複数行コマンドを完走できず、rails runner が空入力で起動 → 何もせず exit 0 → `EventApplication` が DB に作られない。

- Node `child_process.execSync` は Windows ではデフォルトで `cmd.exe /c <command>` を経由する。
- `setupApplication()` が渡すコマンドは template literal に LF を含む multi-line 文字列。
- cmd.exe は引用符内の LF を期待どおり扱えず、最初の行 `docker compose exec -T backend rails runner "` までしか正しく解釈されない。
- それでも `docker compose exec` も `rails runner` も「空入力 → 何もせず exit 0」を返すため、`execSync` は失敗を検知できない。
- 結果: `setupApplication()` は呼ばれているのに `EventApplication` が 1 件も作られず、ダッシュボードは Empty State を表示し、可視性アサーションが全部落ちる。

## 確定させた決め手

`setupApplication` に一時的にロガーを仕込み、コマンド本文・`execSync` の戻り値・直後の DB 件数を同時に取った:

```
[CMD bytes=333]
docker compose exec -T backend rails runner "
      user = User.find_by(email: 'e2e-participant@example.com')
      event = Event.find_by(title: 'E2E公開イベント')
      EventApplication.where(user: user).delete_all
      EventApplication.create!(user: user, event: event, status: :applied, applied_at: Time.current)
    " 2>&1
[CMD END]
[OK out.len=0]   ← throw せず、stdout/stderr とも空
[VERIFY] 0       ← 直後に別コマンドで件数を数えると 0
```

- `create!` が走ったなら必ず 1 件以上残る → `[VERIFY] 0` は **Ruby ブロックが実行されていない** 証拠。
- `out.len=0` は `2>&1` 込みでも空 → Ruby が `RecordInvalid` のような例外を出したわけでもない。
- バックエンドのログでも `EventApplications` クエリは確かに発火し、`COUNT = 0` で正しい応答を返している。DB が真っ当に空。

(検証用のロガーは投稿時点で完全に削除済み。`setupApplication` は元の状態に戻してあります)

## 矛盾しているように見えた点の説明

| 観測 | 説明 |
| --- | --- |
| Empty State テストだけパスする | このテストは `resetApplications()` を呼ぶ。これも同じ原因で空振りしているが、グローバル setup 直後で DB が元々 0 件なので **偶然パスしているだけ** |
| `public-event-apply.spec.ts` 全パス | あちらは `execSync` で複数行 Ruby を流していない (要 spec の setup 確認) |
| 手で `docker compose exec ...` を叩くと動く | ターミナル (PowerShell/Bash) から直接打っているので cmd.exe を経由しない |
| 同僚は気付いていない | おそらく Mac で開発・CI も Linux/Mac。`execSync` のデフォルトシェルが `/bin/sh` の環境では複数行コマンドが通るので、Windows でだけ顕在化する |
| `execSync` が throw しない | `rails runner` が「空のスクリプトを実行して exit 0」を返すため、Node から見ると完全に正常 |

## 切り分けに使った手順 (順番に確証を積んだもの)

1. リゾルバ / スコープ / GraphQL クエリを目検 → 正常。
2. シード状態と `EventApplication` の生成を rails runner で手動再現 → `upcoming: 1` が確実に出る。
3. テスト中の backend ログを採取 → `EventApplications` クエリは `user_id` 正しく発火、`WHERE start_at >= NOW()` も正しい。`COUNT = 0` を返している。
4. `setupApplication` 内で `execSync` の挙動を採取 → exit 0 / 出力空 / DB 件数 0。
5. `docker compose exec -T backend sh -c "echo hi > /tmp/dbg.log"` (シングルライン) はテスト中でも正常動作 → コマンド経路自体は健在。
6. multi-line の `rails runner "..."` だけ何も書かない → **cmd.exe + LF が犯人と確定。**

## 提案: 修正の方向性 (未着手)

| 案 | 内容 | 良さ | 注意点 |
| --- | --- | --- | --- |
| ① shell 明示 | `execSync(cmd, { shell: "bash" })` を全 helper に付ける | 修正範囲が一番小さい。`setupApplication`/`resetApplications` だけ書き換えれば済む | Windows 側に Git Bash があることが前提。CI が Linux なら影響なし |
| ② 一行化 | Ruby を `;` で繋いで 1 行にまとめる | 依存ゼロ | 横長になり可読性が落ちる。エスケープが複雑化 |
| ③ 外部ファイル化 | `db/seeds/e2e_setup_application.rb` を切り出し、`rails runner db/seeds/...` を呼ぶだけにする | シェル経路の差を完全に排除。冪等で再利用しやすい | ファイルが 1〜2 個増える。引数の渡し方を考える必要あり (環境変数 or 引数) |

**推奨**: ③ (恒久対応・他ヘルパーにも横展開できる)。今すぐ通したいだけなら ① が最速。

## 現在のファイル状態

- `e2e/dashboard-applications.spec.ts`: `setupApplication` は **元のコードに完全復元** 済み。git diff に残るのは元々あった Googleマップ用テストケース追加分のみ。
- `packages/frontend/dbg.log`: 削除済み。
- ほかの編集は無し。
