---
title: Playwright E2E テスト作成ガイド（DB 共有・seed・セレクタ・落とし穴）
date: 2026-06-04
tags: [playwright, e2e-testing, seed, selectors, testing]
---

## 概要

Playwright で E2E テストを書くときの全体構成、テストデータ（seed）の扱い、spec ファイルの定型、セレクタ選択の規約、よくある落とし穴、状態が変化するテストの書き方、実行方法、そして teardown スクリプトの役割をまとめたガイド。

## 1. 全体構成・設定

| 項目 | 内容 |
| --- | --- |
| テスト場所 | `packages/frontend/e2e/*.spec.ts` |
| 設定ファイル | `packages/frontend/playwright.config.ts` |
| 対象URL | `http://localhost:3000`（Docker 起動中の Next.js） |
| ブラウザ | Chromium のみ |
| 並列実行 | `workers: 1`（順次実行）← 開発 DB を共有するため |
| リトライ | `retries: 0`（失敗を即検知） |
| タイムアウト | 1 テスト 30 秒 |

⚠️ 最重要：**DB を共有する**
全テストが同じ開発 DB を共有します。だから `workers: 1` で順次実行。テスト間で状態が漏れないよう、データの作り方と順序に注意が必要です。

## 2. テストデータ（seed）の仕組み

```text
globalSetup（テスト全体の前に1回）
  → e2e_teardown.rb（保険で全削除）
  → e2e.rb（データ作成）
globalTeardown（テスト全体の後に1回）
  → e2e_teardown.rb（全削除）
```

- seed は `docker compose exec -T backend rails runner db/seeds/e2e.rb` で実行
- seed が走るのは 1 テストごとではなく「実行全体の最初に 1 回だけ」

### seed 作成の決まり事（`db/seeds/e2e.rb`）

冪等にする（何度流しても同じ結果）。

```ruby
ended_event = Event.find_or_initialize_by(title: "E2E終了イベント")  # 無ければ用意/あれば取得
ended_event.assign_attributes(...)                                  # 属性をまとめてセット
ended_event.save!                                                   # ここで保存
```

`create!` は使わない（実行のたびに重複が増えるため）。

- 参加者は `pm_` 接頭辞 + `e2e-pm-*@example.com`（参加者管理テスト用の慣習）
- 新しい参加者を追加したら `e2e_teardown.rb` の削除リストにも必ず追記（残骸防止）
- 日付は用途に合わせる
  - 開催前の挙動 → `start_at: 1.week.from_now`
  - 終了後の挙動（No-show・出欠締め）→ `end_at: 1.day.ago`
- Event モデルにバリデーションは無い（検証は mutation 側）ので、過去日時の published イベントも seed で直接保存できる

## 3. spec ファイルの定型

```ts
import { test, expect, type Page } from "@playwright/test";

// 認証情報・テストデータ名は定数化
const ORGANIZER = { email: "e2e-test@example.com", password: "password1" };
const EVENT_TITLE = "E2E公開イベント";
const PARTICIPANT = "E2E参加者A";

// ログインヘルパー
async function login(page: Page) {
  await page.goto("/login");
  await page.getByLabel("メールアドレス").fill(ORGANIZER.email);
  await page.getByLabel("パスワード").fill(ORGANIZER.password);
  await page.getByRole("button", { name: "ログイン" }).click();
  await expect(page).toHaveURL("/dashboard");
}

// 画面遷移ヘルパー（引数で対象を切り替えられるように）
async function goToParticipantsTab(page: Page, eventTitle: string) {
  await login(page);
  await page.goto("/organizers/events");
  await page.getByText(eventTitle).first().click();
  await expect(page).toHaveURL(/\/organizers\/events\/\d+/);
  await page.getByRole("tab", { name: "参加者" }).click();
}

test.describe("グループ名", () => {
  test("◯◯できる", async ({ page }) => {
    await goToParticipantsTab(page, EVENT_TITLE);
    // ...
  });
});
```

## 4. セレクタの規約（推奨優先順）

role ベースを優先（アクセシブルで壊れにくい）:

| 用途 | 書き方 |
| --- | --- |
| 行を特定 | `page.getByRole("row", { name: new RegExp(PARTICIPANT) })` |
| ボタン | `page.getByRole("button", { name: "受付する" })` |
| 見出し（ダイアログ等） | `page.getByRole("heading", { name: "参加取消の確認" })` |
| ラジオ | `page.getByRole("radio", { name: /参加完了した人のみ/ })` |
| 入力（label 付き） | `page.getByLabel("件名")` |
| 入力（placeholder） | `page.getByPlaceholder("名前・電話番号で検索")` |
| テキスト | `page.getByText(/順次送信します/)` |

### 行スコープで曖昧さを消す

同じボタンが各行にあるので、行を取ってからその中のボタンを押す:

```ts
await page
  .getByRole("row", { name: new RegExp(PARTICIPANT) })
  .getByRole("button", { name: "無断欠席として記録" })
  .click();
```

## 5. よくある落とし穴と対処

| 落とし穴 | 対処 |
| --- | --- |
| ボタンの aria-label とダイアログ見出しが同名（例「無断欠席として記録」） | 見出しは `getByRole("heading", {...})`、ボタンは `getByRole("button", {...})` で役割を分けて特定 |
| MUI のラベルに件数が含まれる（「全員（3 名）」） | `getByRole("radio", { name: /全員/ })` のように正規表現で部分一致 |
| ダイアログが 2 枚開く（フォーム＋確認） | `page.getByRole("dialog").filter({ hasText: "送信の確認" })` でスコープ |
| disabled ボタンの検証 | `await expect(locator).toBeDisabled()` |
| 件数に依存した assertion | 避ける。イベント名 + `.first()` で対象を取る（seed 追加に強い） |
| テキストの重複（サイドバーのメニュー等） | `page.getByRole("table").getByText("参加予定").first()` のようにコンテナでスコープ |

## 6. 状態が変化するテスト（シリアル）

DB を共有し seed は最初の 1 回だけなので、状態を順に変えるシナリオは `test.describe.serial` でまとめ、実行順に依存させる:

```ts
test.describe.serial("ステータス変更（シリアル）", () => {
  test("applied → attended", async ({ page }) => { /* 受付 */ });
  test("attended → applied", async ({ page }) => { /* 戻す */ });
  test("applied → rejected", async ({ page }) => { /* 取消 */ });
});
```

### 破壊的操作は専用データで隔離する

一括処理（出欠締め＝applied 全員を no_show 化）など他テストに影響する操作は、専用イベント・専用参加者を seed に用意して隔離する（例: E2E 締めイベント + E2E 締め参加者）。

## 7. 実行方法

前提：Docker スタック（backend / db / worker）と Next.js 開発サーバー（`:3000`）が起動中であること。

```bash
# 全テスト
cd packages/frontend && npx playwright test

# 特定ファイルのみ
npx playwright test e2e/organizer-participants-noshow.spec.ts --reporter=list

# パースだけ確認（サーバー不要・テスト検出のみ）
npx playwright test <file> --list
```

プロジェクトには `/e2e` スキルもあり、これ経由でも実行できます。

## `e2e_teardown.rb` とは

### ひとことで

E2E テスト用に作ったデータを全部削除するスクリプト。テストを毎回まっさらな状態から始める（＆終わったら後片付けする）ための掃除役です。

### いつ実行されるか

2 か所から呼ばれます（`global-setup.ts` / `global-teardown.ts`）:

```text
globalSetup（テスト開始前）
  → e2e_teardown.rb  ← ① 前回の残骸を掃除（保険）
  → e2e.rb           ← その後きれいな状態で seed 作成

globalTeardown（テスト終了後）
  → e2e_teardown.rb  ← ② 後片付け
```

つまり「**始める前**」と「**終わった後**」の両方で掃除しています（保険として二重に）。

### 何を削除しているか

```ruby
ORGANIZER_EMAIL   = "e2e-test@example.com"
PARTICIPANT_EMAIL = "e2e-participant@example.com"
```

メールアドレスで対象ユーザーを特定し、関連データを順に削除します:

| 対象 | 削除内容 |
| --- | --- |
| 主催者ユーザー | 施設に紐づくイベントの申込 → イベント → 施設 → 主催者 → ユーザーの順で削除 |
| 参加者ユーザー | 申込 → participant プロフィール → ユーザー |
| 参加者管理テスト用（`e2e-pm-*`） | 申込 → participant → ユーザー |

### 削除順序が大事な理由

外部キー制約があるため、子から親へ順に消します。例えばコメントにもある通り:

```ruby
# participants テーブルの外部キー制約で users 削除が失敗するため、profile を先に削除
participant_user.participant&.delete
participant_user.delete
```

→ いきなりユーザーを消すと「参加者プロフィールが参照してるよ」と制約エラーになるので、先にプロフィールを消してからユーザーを消す。同様にイベントも「申込 → イベント → 施設」と子から順に消しています。

## ポイント

- DB を共有するため `workers: 1` で順次実行・`retries: 0` で失敗を即検知
- seed は実行全体の最初に 1 回だけ走るので、冪等に書く（`find_or_initialize_by` を使い `create!` は避ける）
- セレクタは role ベース優先、行スコープで曖昧さを消す、表示テキストは正規表現で部分一致
- 同名のボタンとダイアログ見出しは役割（button / heading）で分けて特定
- 状態を順に変えるテストは `test.describe.serial` でまとめ、破壊的操作は専用 seed データで隔離
- `e2e_teardown.rb` は外部キー制約に従い「子 → 親」の順で削除し、setup と teardown の両方で実行して二重に掃除する
