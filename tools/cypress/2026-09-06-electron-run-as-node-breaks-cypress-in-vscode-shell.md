---
title: VSCode 発のシェルで Cypress が起動しないのは ELECTRON_RUN_AS_NODE の継承が原因
date: 2026-09-06
tags: [cypress, electron, vscode, e2e]
source: 実務(Next.js)
---

## 概要

VSCode 拡張が開く子シェルから `npx cypress run` を実行すると、`bad option: --no-sandbox` で起動に失敗する。VSCode 自身が Electron 製で、拡張ホストの環境変数 `ELECTRON_RUN_AS_NODE=1` が子シェルへ継承され、Electron 製バイナリの Cypress が「素の Node」として起動してしまうのが原因。環境変数を外して実行するだけで直る。

## サンプルコード

```sh
# 解決: ELECTRON_RUN_AS_NODE を外して実行する（unset でもよい）
env -u ELECTRON_RUN_AS_NODE npx cypress run
```

```sh
# 診断: バイナリに --version を聞いて Node のバージョンが返れば、Node として起動している証拠
"$HOME/Library/Caches/Cypress/<ver>/Cypress.app/Contents/MacOS/Cypress" --version
# => v16.16.0（Cypress ではなく Node のバージョン）
```

## ポイント

- Electron 製バイナリは `ELECTRON_RUN_AS_NODE=1` があると Node モードで起動する仕様（Electron 公式の環境変数）。Node は `--no-sandbox` や `--smoke-test` などの Chromium/Electron 用フラグを解釈できず `bad option` で終了するため、Cypress の起動検証（smoke test）の段階で失敗する。
- `npx cypress install --force` での再ダウンロードでは直らない（バイナリ破損ではないため）。「バイナリに `--version` を聞いて Node のバージョンが返る」が数秒で切り分けられる診断法。
- Cypress 固有の問題ではない。Electron を内包する CLI ツール全般で、VSCode 発のシェルから実行すると同型の問題が起こり得る。「Electron 製ツールが Node として振る舞う」症状を見たら、まず `ELECTRON_RUN_AS_NODE` を疑う。
- 付随の学び: ヘッドレス E2E でマイク/カメラ（getUserMedia）を使う画面を通すには、`ELECTRON_EXTRA_LAUNCH_ARGS="--use-fake-ui-for-media-stream --use-fake-device-for-media-stream"` を環境変数で渡すと、`cypress.config` の `setupNodeEvents` を書き換えずに OS の権限ダイアログを避けられる。
- 付随の学び: テスト対象外のビデオ通話 SDK の接続失敗でテストが落ちる場合は、スペック内で `Cypress.on("uncaught:exception", () => false);` として無関係な例外を無視できる。

## 参考

- Electron 公式ドキュメント「Environment Variables」（`ELECTRON_RUN_AS_NODE`）: https://www.electronjs.org/docs/latest/api/environment-variables
- Cypress 公式ドキュメント「Browser Launch API」（ブラウザ起動引数・`ELECTRON_EXTRA_LAUNCH_ARGS`）: https://docs.cypress.io/api/plugins/browser-launch-api
