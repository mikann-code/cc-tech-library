---
title: 3段階トークンフロー — メール認証で本データを更新する設計パターン
date: 2026-05-18
tags: [security, authentication, design-pattern, email-verification, token]
---

## 概要

「メールアドレス変更」「パスワードリセット」「アカウント削除確認」のように、**操作の主体が本人であることをメール経由で確認してから本データを更新する**機能は、多くのアプリで必要になる。

教科書的には「mutation を1本叩いて更新」だが、実運用ではそれだとさまざまな事故が起きる。本章では「リクエスト → 検証 → 確定」の3段階に分割するパターンと、その各段階で何を達成すべきかを説明する。

## サンプルコード

### ナイーブな実装（問題あり）

```ruby
# ❌ ナイーブな実装
class UpdateEmail < BaseMutation
  argument :new_email, String, required: true

  def resolve(new_email:)
    user = context[:current_user]
    user.update!(email: new_email)
    { success: true }
  end
end
```

これだけでは以下の事故が起きる:

- **「宙ぶらりん状態」が発生する**: `email` カラムを直接書き換えると、旧メアドではもうログインできず、新メアドは本人が本当に所有しているか未検証になり、入力ミスで「自分のアカウントにログインできない」事故が起きる。
- **本人確認がない**: ログイン状態を維持したまま離席した端末を使われると、第三者が任意のメアドに書き換えてアカウントを乗っ取れる。
- **メーラーのプリフェッチで誤確定する**: 「メールにリンクを貼って、開いたら確定」という素朴なフローだと、Outlook / Gmail のセキュリティスキャナーや、URL プレビュー機能で**ユーザーがクリックする前に URL がアクセス**される。GET リクエストで状態が変わると、ここで誤確定する。
- **React Strict Mode の二重発火**: Next.js などの開発環境では `useEffect` が2回走る。確定処理を `useEffect` 内で叩くと2回実行される。

### 3段階フローの全体像

```
[Step 1] リクエスト（requestEmailChange）
  ・本人確認（現パスワードの照合）
  ・新メアドのバリデーション（形式・重複・現メアドとの同一性）
  ・トークン生成 + pending_email カラムに退避
  ・確認メール送信
  ↓ ユーザーがメール内リンクをクリック
[Step 2] 検証（confirmEmailChangeToken）
  ・トークンの有効性チェック（存在・期限）
  ・新メアドを画面に表示「○○ に変更しますか？」
  ・★ DB は変更しない（副作用なし）
  ↓ ユーザーが「変更を確定する」ボタンを押下
[Step 3] 確定（completeEmailChange）
  ・もう一度トークン検証
  ・email カラムを pending_email の値で書き換え
  ・pending_email / token をクリア
```

各段階の役割分担を明確に分けることで、それぞれの「失敗モード」を独立に防げる。

### Step 1: リクエスト（`request*`）

**やること:**
- 本人確認（パスワード再入力）
- 新値のバリデーション（形式・重複・「現在の値と同じ」拒否）
- ランダムトークン生成（`SecureRandom.urlsafe_base64(32)` 程度）
- `pending_*` カラムに新値を退避（本データはまだ書き換えない）
- 確認メール送信

```ruby
def resolve(current_password:, email:)
  user = context[:current_user]
  errors = validate(user:, current_password:, email:)
  return { success: false, errors: errors } if errors.any?

  # 本データ（user.email）は書き換えず、pending_email に退避
  user.update!(
    pending_email: email,
    email_change_token: SecureRandom.urlsafe_base64(32),
    email_change_token_sent_at: Time.current,
  )

  EmailChangeMailer.email_change_email(user).deliver_later
  { success: true, errors: [] }
end
```

DB スキーマには `pending_email`, `email_change_token`, `email_change_token_sent_at` の3カラムを追加する。

### Step 2: 検証（`*Token`）

**やること:**
- トークンの存在チェック
- 有効期限チェック（24時間など）
- 「○○ に変更しますか？」画面に表示する新メアドを返す
- **DB は一切変更しない**

```ruby
def resolve(token:)
  user = User.find_by(email_change_token: token)

  return error("無効なトークンです") if user.nil?
  return error("期限切れです") if user.email_change_token_sent_at < 24.hours.ago

  # ★ 副作用なし。検証結果と新メアドを返すだけ。
  { success: true, errors: [], email: user.pending_email }
end
```

**この副作用なし設計が重要**な理由:

| 副作用あり | 副作用なし |
|---|---|
| メーラーのプリフェッチで誤確定 | プリフェッチされても安全 |
| Strict Mode 二重発火で2回確定 | 何度呼ばれても結果同じ |
| リロードで誤確定 | リロード自由 |
| 多重タブで誤確定 | 多重タブ可 |

「検証用 mutation を副作用なしに保つ」ことが、3段階フローの肝。

### Step 3: 確定（`complete*`）

**やること:**
- もう一度トークン検証（Step 2 から時間が経っている可能性）
- 本データを `pending_*` の値で書き換え
- `pending_*` / トークンをクリア

```ruby
def resolve(token:)
  user = User.find_by(email_change_token: token)
  return error("無効なトークンです") if user.nil?
  return error("期限切れです") if user.email_change_token_sent_at < 24.hours.ago

  user.update!(
    email: user.pending_email,
    pending_email: nil,
    email_change_token: nil,
    email_change_token_sent_at: nil,
  )
  { success: true, errors: [] }
end
```

確定は **ユーザーの明示的なクリック**（フロントの「変更を確定する」ボタン）でのみ呼ぶ。`useEffect` で自動実行しない。

### フロントエンドの構造

「検証」と「確定」を別ページにせず、**1ページ内で4ステートに出し分け**るのがシンプル:

```tsx
function ConfirmEmailChangePage() {
  const [verifying, setVerifying] = useState(true);
  const [pendingEmail, setPendingEmail] = useState<string | null>(null);
  const [applied, setApplied] = useState(false);
  const [errorMessage, setErrorMessage] = useState<string | null>(null);

  // ページマウント時にトークン検証（副作用なし mutation）
  useEffect(() => {
    confirmToken(token).then(res => {
      if (res.success) setPendingEmail(res.email);
      else setErrorMessage(res.errors[0].message);
      setVerifying(false);
    });
  }, [token]);

  // ユーザーのクリックで初めて確定 mutation を叩く
  const handleConfirm = async () => {
    const res = await completeChange(token);
    if (res.success) setApplied(true);
  };

  if (verifying) return <Spinner />;
  if (errorMessage) return <ErrorView message={errorMessage} />;
  if (applied) return <CompletedView />;
  return <ConfirmView email={pendingEmail} onConfirm={handleConfirm} />;
}
```

| ステート | 表示 | トリガー |
|---|---|---|
| `verifying` | スピナー | 初期状態 |
| `errorMessage` あり | エラー画面 | トークン無効・期限切れ |
| `applied` | 完了画面 | 確定 mutation 成功 |
| それ以外 | 確認画面 | 検証成功・確定待ち |

## 命名規則とパターンの一貫性

3段階の各 mutation を統一的な命名にすることで、コードベース全体での mental model が揃う:

| 段階 | プレフィックス/サフィックス | 例 |
|---|---|---|
| リクエスト | `request*` | `requestEmailChange`, `requestPasswordReset` |
| 検証 | `*Token` | `confirmEmailChangeToken`, `verifyPasswordResetToken` |
| 確定 | `complete*` | `completeEmailChange`, `completePasswordReset` |

**新規ユーザー登録の本登録フロー**（仮登録 → メール認証 → 本登録）もこの3段階に揃えると、認証絡みの mutation がすべて同じパターンになり、新メンバーのオンボーディングが楽になる。

## このパターンを使うべきとき・使わないとき

### 使うべき
- メールアドレス変更
- パスワードリセット（ログイン不可能な状態からの回復）
- アカウント削除の最終確認
- 重要な権限変更（管理者昇格など）

### 使わないでよい
- 名前・電話番号など、本人確認の影響が小さい属性の変更
- ログイン状態でのパスワード変更（現パス入力で本人確認は完結する）
- 公開プロフィールの編集

「**操作後に本人確認のチャネルが失われるか？**」が判断基準。メアド変更は変更後に旧メアドが使えなくなるので、変更前に本人確認チャネル経由で「本当にこの新メアドでいいか」を取り直す必要がある。

## ありがちな失敗パターン

### 検証 mutation に副作用を入れてしまう

「トークンを使ったら無効化したい」と思って `token = nil` するなど。**確定 mutation でクリアすればよい**。検証は何度呼んでも同じ結果を返す純粋関数にする。

### 確定を `useEffect` 内で自動実行する

「リンクをクリックしたら自動で完了」したい気持ちはわかるが、プリフェッチ・二重発火・リロードで全部壊れる。**確定はユーザーのクリック**でのみ。

### トークンに有効期限を持たせない

24時間ほどの有効期限を持たせ、検証・確定の両方で期限チェックする。古いリンクで誤確定するのを防ぐ。

### リクエスト時に本人確認を省く

「ログイン中だから本人だろう」と省くと、離席した端末から乗っ取られる。**ログイン中でも、本データを書き換える操作には現パスワード再入力**を要求する（セッションの古さでバイパス判定する設計もある）。

### `pending_*` カラムを作らず、直接 `email` を更新してしまう

「現メアドでログイン不可・新メアドも未認証」の宙ぶらりん状態が発生する。**確定するまで本データは絶対に変えない**。

## まとめ

| 段階 | 副作用 | DB 書き換え | トリガー |
|---|---|---|---|
| リクエスト | ✅ ある（`pending_*` に退避） | 部分的 | フォーム送信 |
| 検証 | ❌ なし | しない | ページマウント（自動） |
| 確定 | ✅ ある（本データ書き換え） | する | ユーザーの明示クリック |

3段階に分けることで、**各段階の失敗モードを独立して防げる**。これが「mutation を1本で済ます」ナイーブ実装との本質的な違いである。

認証・本人確認絡みの機能を設計するときは、まずこの3段階フローに当てはまるかを検討してみると、多くの落とし穴を未然に防げる。

## ポイント

- 「リクエスト → 検証 → 確定」の3段階に分割し、それぞれの失敗モードを独立に防ぐ。
- リクエスト段階では本データ（`email`）を書き換えず、`pending_email` に退避して「宙ぶらりん状態」を防ぐ。
- 検証段階の mutation は**副作用なし**に保つ。これによりメーラーのプリフェッチ、Strict Mode 二重発火、リロード、多重タブのいずれでも誤確定しない。
- 確定段階はユーザーの明示的なクリックでのみ呼ぶ。`useEffect` で自動実行しない。
- リクエスト時は本人確認（現パスワード照合）を行い、ログイン状態を悪用したアカウント乗っ取りを防ぐ。
- トークン検証は Step 2 と Step 3 の両方で行う（Step 2 から時間が経っている可能性があるため）。
- フロントは1ページ内で「検証中 / エラー / 完了 / 確認待ち」の4ステートに出し分けるとシンプル。
- 命名は `request*` / `*Token` / `complete*` で統一すると、認証絡み mutation の mental model が揃う。
- 判断基準は「操作後に本人確認のチャネルが失われるか？」。失われるならこのパターンを使う。
