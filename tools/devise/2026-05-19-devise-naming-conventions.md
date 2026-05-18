---
title: Devise 慣例（カラム名・テーブル設計の命名規則）
date: 2026-05-19
tags: [devise, rails, authentication, naming-convention]
---

## 概要

Devise（Ruby on Rails で広く使われる認証用 Gem）が採用しているカラム名・テーブル設計の命名慣習のこと。Rails コミュニティでは事実上の標準になっており、Devise 自体を使わなくてもこの命名に従うと役割が一目で分かるというメリットがある。

## 代表的なカラム名

### パスワード関連

| カラム名 | 役割 |
| --- | --- |
| `encrypted_password` | bcrypt でハッシュ化したパスワード（生のパスワードは保存しない） |
| `reset_password_token` | パスワードリセット用トークン |
| `reset_password_sent_at` | リセットメール送信日時（有効期限判定に使用） |
| `remember_created_at` | 「ログイン状態を保持」を有効化した日時 |

### ログイン履歴（Trackable）

| カラム名 | 役割 |
| --- | --- |
| `sign_in_count` | ログイン回数 |
| `current_sign_in_at` / `last_sign_in_at` | 現在 / 前回のログイン日時 |
| `current_sign_in_ip` / `last_sign_in_ip` | 現在 / 前回のログイン IP |

### メール確認（Confirmable）

| カラム名 | 役割 |
| --- | --- |
| `confirmation_token` | メールアドレス確認用トークン（仮登録→本登録） |
| `confirmed_at` | メール確認完了日時 |
| `confirmation_sent_at` | 確認メール送信日時 |
| `unconfirmed_email` | メール変更時の「新メアド（未確認）」を一時保存 |

### アカウントロック（Lockable）

| カラム名 | 役割 |
| --- | --- |
| `failed_attempts` | 連続ログイン失敗回数（ロック判定用） |
| `unlock_token` | アカウントロック解除トークン |
| `locked_at` | ロック日時 |

## このプロジェクトとの関係

このプロジェクトは Devise を使っていない（DB ベースのセッション + 自前の Mutation）。ただし、`users` テーブルや本登録トークン周りのカラム名を Devise 慣例に寄せておくと、以下のメリットがある。

- 他の Rails エンジニアが見たときに役割が一瞬で分かる
- 将来 Devise / devise-token-auth に乗せ替える際の差分が小さい
- `confirmation_token` / `reset_password_token` のようにセキュリティ的に枯れたパターンを踏襲できる

## ポイント

- Devise の命名慣例は Rails コミュニティで事実上の標準。Devise 自体を使わなくても、命名に従うと役割が一目で分かる。
- カラムは機能別グループ（パスワード / Trackable / Confirmable / Lockable）でまとまっている。
- トークン系は `*_token` + `*_sent_at` の2セットで「有効期限つきトークン」を表現する。
- メール変更時の「新メアド（未確認）」は `unconfirmed_email` という別カラムに退避する（本データは確認後に書き換え）。
- Devise を使わない自前実装でも、命名を寄せておくと将来 Devise / devise-token-auth に乗せ替える際の差分が小さい。
