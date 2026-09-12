---
title: Flutter に「ページ」という型はない — 画面も部品も余白もすべて Widget であることを継承の連鎖とルーター定義で確かめた
date: 2026-09-12
tags: [flutter, widget, riverpod, go-router, dart]
source: 実務(Flutter)
---

# Flutter に「ページ」という型はない — 画面も部品も余白もすべて Widget であることを継承の連鎖とルーター定義で確かめた

## 概要

React / TypeScript / Rails の経験はあるが Flutter は初めて、という状態でモバイルアプリのコードを読み始め、`class SettingsPage extends HookConsumerWidget` の継承チェーンを ~/.pub-cache の実ソースで Widget まで辿った。go_router の `builder` は「Widget を返す関数」を受け取るだけであることも確認し、「Page = 名前の付いたただの Widget」「余白や中央寄せも Widget」という Flutter の前提を React / TypeScript との対比で整理した。

## サンプルコード

```dart
// pages/settings_page.dart — 「ページ」もただの Widget
class SettingsPage extends HookConsumerWidget {
  const SettingsPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return const Scaffold(
      body: SingleChildScrollView(
        child: SettingsBody(),
      ),
    );
  }
}

// provider/router.dart — builder は「Widget を返す関数」を受け取る口でしかない
GoRoute(
  path: '/settings',
  builder: (_, __) => const SettingsPage(),
),
```

## ポイント

- Flutter には「ページ」を特別扱いする型は存在しない。`extends HookConsumerWidget` は「Widget の仲間に入るので Widget ツリーのどこにでも置け、build で自分の中身を描ける」という宣言で、継承チェーンは HookConsumerWidget（hooks_riverpod）→ ConsumerWidget（flutter_riverpod）→ ConsumerStatefulWidget → StatefulWidget → Widget と必ず Widget に行き着く。各層が足すのは順に「hooks が使える」「build の第2引数 `ref`」「Riverpod への接続」「状態と再描画」。見慣れないクラスは Cmd+クリック（F12）で定義に飛び、~/.pub-cache の実ソースを Widget まで辿るのが早い
- pubspec.yaml に hooks_riverpod しか書いていなくても ConsumerWidget や ProviderScope が使えるのは、入口ファイル lib/hooks_riverpod.dart が `export 'package:flutter_riverpod/flutter_riverpod.dart';` で丸ごと再公開しているため（npm の `export * from` 相当）。対応関係は pubspec.yaml = package.json、pubspec.lock = package-lock.json、~/.pub-cache（マシン全体で共有）= node_modules
- 「ページかどうか」はルート定義の `builder` が返しているかどうかだけの違いで、型は変わらない（`Text` を渡しても型としては通る）。ボトムナビ付きレイアウトは ShellRoute の builder で外枠に包み、その `child` にページ Widget が差し込まれる。React Router の `<Route element={<SettingsPage />} />` や Next.js の pages/ が普通のコンポーネントを export するのと同じ構造
- CSS で書いていた余白・中央寄せ・サイズも Widget になる: `Padding` ⇔ `padding`、`Center` ⇔ flex 中央寄せ、`SizedBox(height: 20)` ⇔ 高さ 20 の空 div、`Column` / `Row` ⇔ flex-direction、`Expanded` ⇔ `flex: 1`、`Divider` ⇔ `<hr />`。子1つなら `child:`、複数なら `children: [...]` で命名が統一されており、深いネストは「包む」の積み重ねと読めばよい
- 運用判断として、StatelessWidget で足りる薄いページも全ページ HookConsumerWidget で揃えておくと、後で `ref` が必要になっても宣言を変えずに済む
