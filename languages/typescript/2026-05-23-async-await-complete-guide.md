---
title: async/await 完全ガイド - 実務で使う非同期処理のすべて
date: 2026-05-23
tags: [typescript, javascript, async, await, promise, asynchronous]
source: study-note
---

## はじめに

`async/await` は Promise を同期的なコードのように書ける構文。非同期処理のコールバック地獄や `.then()` チェーンを解消し、可読性とエラーハンドリングを劇的に改善する。

実務では **API 呼び出し、DB アクセス、ファイル I/O、外部サービス連携** など、I/O を伴う処理のほぼすべてで使う。正しく使えないとパフォーマンス問題、エラー握りつぶし、メモリリーク、デッドロックなどに直結するため、本質を理解することが重要。

---

## 1. 前提: Promise の基本

`async/await` は Promise のシンタックスシュガー。仕組みを理解するには Promise から始める。

### Promise の 3 つの状態

```ts
// 1. pending  : まだ完了していない
// 2. fulfilled: 成功して値が確定した
// 3. rejected : 失敗してエラーが確定した

const promise = new Promise<string>((resolve, reject) => {
  setTimeout(() => {
    if (Math.random() > 0.5) {
      resolve("成功");        // pending → fulfilled
    } else {
      reject(new Error("失敗")); // pending → rejected
    }
  }, 1000);
});
```

一度確定した状態は **二度と変わらない**（不変）。これが Promise の重要な性質。

### `.then` / `.catch` / `.finally`

```ts
fetchUser(1)
  .then((user) => console.log(user))      // fulfilled 時
  .catch((err) => console.error(err))     // rejected 時
  .finally(() => console.log("done"));    // 成否に関係なく実行
```

実務でこのチェーンを長く書くと**ネストや分岐で読めなくなる**。そこで `async/await` の出番。

---

## 2. `async` 関数の基本

### `async` を付けると何が起きるか

```ts
async function getUser(id: number): Promise<User> {
  // ...
  return user;
}
```

`async` を付けた関数は **必ず Promise を返す**。

- `return user` と書いても、実際は `Promise.resolve(user)` が返る
- `throw new Error(...)` と書いても、実際は `Promise.reject(...)` が返る

つまり戻り値の型は常に `Promise<T>` になる。

```ts
async function getNumber(): Promise<number> {
  return 42;  // Promise<number> として返る
}

// 呼び出し側
getNumber().then((n) => console.log(n)); // 42
```

### アロー関数版

```ts
const getUser = async (id: number): Promise<User> => {
  // ...
  return user;
};
```

クラスメソッドでも書ける。

```ts
class UserService {
  async getUser(id: number): Promise<User> {
    return await this.repository.findById(id);
  }
}
```

---

## 3. `await` の動作

`await` は **Promise が解決されるまで関数の実行を一時停止** し、解決された値を返す。

```ts
async function main() {
  const user = await fetchUser(1);  // Promise<User> → User
  console.log(user.name);
}
```

### 重要な特性

- `await` は **`async` 関数の中でしか使えない**（トップレベル `await` は ES2022 のモジュールで例外的に可）
- `await` する値が Promise でなければ、そのまま即座に値が返る
- `await` の前後で **関数の実行が中断される**（イベントループに制御が戻る）

### `await` の裏側で何が起きているか

```ts
async function example() {
  console.log("A");
  const result = await fetchSomething();  // ここで関数が一時停止
  console.log("B"); // Promise が解決されてから実行される
  return result;
}

console.log("1");
example().then((r) => console.log("3", r));
console.log("2");

// 出力順:
// 1
// A
// 2
// (fetchSomething 完了後)
// B
// 3 <result>
```

`await` は実質的に **「ここで関数の続きをコールバックに登録して return する」** ことに等しい。

---

## 4. エラーハンドリング: `try/catch`

`.catch()` の代わりに `try/catch` が使える。これが `async/await` の最大のメリット。

```ts
async function loadUserProfile(id: number) {
  try {
    const user = await fetchUser(id);
    const posts = await fetchPosts(user.id);
    return { user, posts };
  } catch (err) {
    console.error("ユーザー情報の取得に失敗:", err);
    throw err;  // 上位に再スロー
  }
}
```

### 実務で重要: エラーの「再スロー」と「握りつぶし」

```ts
// ❌ 悪い例: エラーを握りつぶしている
async function bad() {
  try {
    return await riskyOperation();
  } catch (err) {
    console.log(err);  // ログだけ出して何も返さない → undefined が返る
  }
}

// ✅ 良い例: 適切に再スローまたは代替値を返す
async function good() {
  try {
    return await riskyOperation();
  } catch (err) {
    logger.error("riskyOperation failed", { err });
    throw new ServiceError("操作に失敗しました", { cause: err });
  }
}
```

### `catch` の型 (TypeScript 4.4+)

```ts
try {
  await something();
} catch (err) {
  // err は unknown 型(デフォルト)
  if (err instanceof NetworkError) {
    // ネットワークエラー固有の処理
  } else if (err instanceof Error) {
    console.error(err.message);
  } else {
    console.error("Unknown error", err);
  }
}
```

`unknown` 型なので、必ず型ガードしてから使う。`any` でキャッチするのは避ける。

---

## 5. 直列実行 vs 並列実行（実務で最も重要）

これを理解していないと **パフォーマンスが致命的に悪化** する。

### ❌ 直列実行 (遅い)

```ts
async function getDashboardData(userId: number) {
  const user = await fetchUser(userId);       // 100ms 待つ
  const posts = await fetchPosts(userId);     // さらに 100ms 待つ
  const followers = await fetchFollowers(userId); // さらに 100ms 待つ
  // 合計: 300ms
  return { user, posts, followers };
}
```

3 つの API が **互いに依存していない** のに、1 つずつ待っている。

### ✅ 並列実行 (速い)

```ts
async function getDashboardData(userId: number) {
  const [user, posts, followers] = await Promise.all([
    fetchUser(userId),
    fetchPosts(userId),
    fetchFollowers(userId),
  ]);
  // 合計: 100ms (最も遅いものに律速)
  return { user, posts, followers };
}
```

### いつ直列か、いつ並列か？

| ケース | 選択 |
|---|---|
| 後続処理が前の結果に依存する | 直列 |
| 互いに独立している | 並列 |
| 一つ失敗したら全部止めたい | `Promise.all` |
| 全部の結果を取得したい（失敗も含む） | `Promise.allSettled` |
| 一番速いものだけ使いたい | `Promise.race` |

---

## 6. `Promise.all` 系メソッド完全比較

### `Promise.all` - 全部成功で完了、一つでも失敗で即終了

```ts
const [a, b, c] = await Promise.all([taskA(), taskB(), taskC()]);
```

**注意**: 一つでも reject されると、その時点で `Promise.all` 全体が reject される。
他のタスクの結果は捨てられる（が、タスク自体は **キャンセルされない**。実行は続く）。

### `Promise.allSettled` - 全部の結果を待つ

```ts
const results = await Promise.allSettled([taskA(), taskB(), taskC()]);

for (const r of results) {
  if (r.status === "fulfilled") {
    console.log("成功:", r.value);
  } else {
    console.error("失敗:", r.reason);
  }
}
```

**実務でよく使うパターン**: 複数の外部 API を叩いて、一部失敗しても処理を続けたい場合。

```ts
// 例: 複数の通知チャネルに送信し、一つ失敗しても他は送る
const results = await Promise.allSettled([
  sendSlack(msg),
  sendEmail(msg),
  sendSMS(msg),
]);

const failed = results.filter((r) => r.status === "rejected");
if (failed.length > 0) {
  logger.warn("一部の通知チャネルで失敗", { failed });
}
```

### `Promise.race` - 最初に確定したもの

```ts
const winner = await Promise.race([taskA(), taskB(), taskC()]);
```

**実務でよく使うパターン**: タイムアウト実装

```ts
function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error(`Timeout after ${ms}ms`)), ms)
  );
  return Promise.race([promise, timeout]);
}

// 使用例: 5秒以内に応答がなければエラー
const data = await withTimeout(fetchData(), 5000);
```

### `Promise.any` - 最初に成功したもの

```ts
const firstSuccess = await Promise.any([taskA(), taskB(), taskC()]);
```

全部失敗すると `AggregateError` が投げられる。
**実務でよく使うパターン**: フェイルオーバー、複数の CDN/ミラーから最速で取得など。

---

## 7. 実務でよく使うパターン集

### 7-1. リトライ処理

```ts
async function retry<T>(
  fn: () => Promise<T>,
  retries = 3,
  delayMs = 1000
): Promise<T> {
  let lastError: unknown;
  for (let i = 0; i < retries; i++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err;
      if (i < retries - 1) {
        await sleep(delayMs * Math.pow(2, i)); // 指数バックオフ
      }
    }
  }
  throw lastError;
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

// 使用例
const data = await retry(() => fetchFlakyApi(), 5, 500);
```

### 7-2. 順次処理 (for-of + await)

配列の各要素を **順番に** 非同期処理したい場合。

```ts
async function processAllInOrder(ids: number[]) {
  for (const id of ids) {
    await processItem(id);  // 1つずつ順番に
  }
}
```

**`forEach` は使ってはいけない**:

```ts
// ❌ awaitが効かない。並列に走り、しかも待たれない
ids.forEach(async (id) => {
  await processItem(id);
});
```

`forEach` は async 関数を渡しても **Promise を待たない**。`for-of` を使う。

### 7-3. 並列処理 + 同時実行数の制限

何百件もの API を一気に並列でぶん投げるとレート制限に引っかかったり、メモリが溢れたりする。

```ts
import pLimit from "p-limit";

const limit = pLimit(5); // 同時実行 5 件まで

const results = await Promise.all(
  ids.map((id) => limit(() => fetchItem(id)))
);
```

`p-limit` ライブラリは実務でほぼ必須。手書きで実装することも可能だが、ライブラリに任せた方が安全。

### 7-4. 順次 reduce で結果を累積する

```ts
async function processSequentially<T, R>(
  items: T[],
  fn: (acc: R, item: T) => Promise<R>,
  initial: R
): Promise<R> {
  let acc = initial;
  for (const item of items) {
    acc = await fn(acc, item);
  }
  return acc;
}
```

### 7-5. async イテレータ (for-await-of)

ストリームやページネーション API で便利。

```ts
async function* paginate(url: string) {
  let nextUrl: string | null = url;
  while (nextUrl) {
    const res = await fetch(nextUrl);
    const data = await res.json();
    yield data.items;
    nextUrl = data.nextPageUrl;
  }
}

for await (const items of paginate("/api/users")) {
  console.log(items);
}
```

### 7-6. AbortController でキャンセル

```ts
const controller = new AbortController();

async function fetchWithCancel(url: string) {
  try {
    const res = await fetch(url, { signal: controller.signal });
    return await res.json();
  } catch (err) {
    if (err instanceof DOMException && err.name === "AbortError") {
      console.log("キャンセルされました");
      return null;
    }
    throw err;
  }
}

// 別の場所でキャンセル
controller.abort();
```

React の useEffect やユーザー操作のキャンセルで頻出。

---

## 8. よくある落とし穴

### 8-1. `await` の付け忘れ

```ts
async function saveUser(user: User) {
  db.save(user);  // ❌ await を忘れている。Promise が放置される
}
```

完了を待たずに次に進んでしまうため、エラーも捕捉できない（Unhandled Promise Rejection に）。
TypeScript の `no-floating-promises` ESLint ルールで検出できる。**絶対に有効化すべき**。

### 8-2. 不要な `await return`

```ts
// 動作はするが冗長
async function getUser(id: number) {
  return await fetchUser(id);
}

// シンプル: そのまま return すれば Promise が伝播する
async function getUser(id: number) {
  return fetchUser(id);
}
```

ただし **`try/catch` の中では `await return` が必要**。`await` しないと catch されない。

```ts
async function safe() {
  try {
    return await fetchUser(1);  // ✅ await 必須
  } catch (err) {
    return null;
  }
}
```

### 8-3. `async` 関数の中でループに `await` を入れて、並列にしたかったのに直列になる

```ts
// ❌ 直列になる (1つずつ順番)
for (const id of ids) {
  await fetchItem(id);
}

// ✅ 並列にする
await Promise.all(ids.map((id) => fetchItem(id)));
```

### 8-4. コンストラクタを async にできない

```ts
class UserService {
  async constructor() {}  // ❌ 構文エラー
}
```

対策: ファクトリ関数パターン。

```ts
class UserService {
  private constructor(private user: User) {}

  static async create(id: number): Promise<UserService> {
    const user = await fetchUser(id);
    return new UserService(user);
  }
}

const service = await UserService.create(1);
```

### 8-5. Promise を作るとすぐに実行が始まる

```ts
const p = fetchUser(1);  // この時点で fetch が開始される
// ...色々な処理...
const user = await p;    // ここでは値を待つだけ
```

「`await` で初めて実行される」ではない。Promise は **作成時に開始** され、`await` は単に結果を待つだけ。
これを利用して **早めに Promise を作って、後でまとめて await する** テクニックがある。

```ts
// 並列で開始させて、必要なときに使う
const userPromise = fetchUser(id);
const settingsPromise = fetchSettings(id);

// 中間処理...

const user = await userPromise;
const settings = await settingsPromise;
```

### 8-6. Unhandled Promise Rejection

Promise を作ったが `.catch` も `await` もしないまま reject されると、Node では警告（将来的にはプロセス終了）、ブラウザではコンソールエラーになる。

```ts
fetchSomething();  // ❌ 戻り値も await も無視
                    // reject されると Unhandled Rejection
```

必ず `await` するか、`.catch()` で処理する。fire-and-forget したい場合は明示的に処理する。

```ts
fetchSomething().catch((err) => logger.error(err));  // ✅ 明示的に握る
```

---

## 9. パフォーマンス・メモリ上の注意

### 並列で大量に走らせない

```ts
// ❌ 10万件並列。メモリ枯渇、API レート制限
await Promise.all(items.map((item) => process(item)));
```

`p-limit` などで同時実行数を制限する。

### ストリーム処理で大量データを扱う

メモリに全部読み込まず、Node.js の Readable Stream や非同期イテレータを使う。

```ts
import { createReadStream } from "fs";
import { createInterface } from "readline";

async function processLargeFile(path: string) {
  const stream = createReadStream(path);
  const rl = createInterface({ input: stream });

  for await (const line of rl) {
    await processLine(line);
  }
}
```

---

## 10. テストでの async/await

```ts
import { describe, it, expect } from "vitest";

describe("UserService", () => {
  it("ユーザーを取得できる", async () => {
    const user = await userService.getUser(1);
    expect(user.id).toBe(1);
  });

  it("存在しないユーザーはエラーを投げる", async () => {
    await expect(userService.getUser(-1)).rejects.toThrow("Not found");
  });
});
```

`rejects.toThrow` を忘れがち。これを使わないと、テストが「Promise を返しただけ」で成功扱いになることがある。

---

## 11. 実務でのチェックリスト

設計・レビュー時に確認すべき項目:

- [ ] `await` の付け忘れはないか？(`no-floating-promises` を有効化)
- [ ] 並列にできる処理が直列になっていないか？(`Promise.all` を検討)
- [ ] 並列実行で同時実行数の上限が必要ではないか？
- [ ] エラーハンドリングは適切か？(握りつぶし・型ガード)
- [ ] タイムアウトは設定されているか？(外部 I/O 全般)
- [ ] リトライ戦略は適切か？(指数バックオフ、最大回数)
- [ ] キャンセル可能にすべきか？(AbortController)
- [ ] ログには十分なコンテキストが含まれているか？
- [ ] テストで Promise の reject 検証ができているか？

---

## 12. まとめ

- `async` 関数は常に `Promise<T>` を返す
- `await` は Promise の解決を待つだけ。**Promise は作った瞬間に開始する**
- エラーは `try/catch` で捕捉。型は `unknown` 前提で型ガード
- **直列か並列か** を常に意識する。並列化できるところは `Promise.all`
- 大量並列は `p-limit` で制限。タイムアウトは `Promise.race`
- 一部失敗を許容するなら `Promise.allSettled`
- `forEach` の中で `await` してはいけない。`for-of` を使う
- フローティング Promise を作らない (`no-floating-promises` 必須)

## 参考

- MDN: [async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)
- MDN: [await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await)
- TypeScript Deep Dive: Async Await
- typescript-eslint: `no-floating-promises`, `await-thenable`
- `p-limit` (同時実行数制限)
