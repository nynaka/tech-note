---
title: Antigravity CLI
description: Google Antigravity CLI のインストールに関する手順です。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

[Antigravity CLI](https://github.com/google-antigravity/antigravity-cli)
===

Antigravity CLI は、Google が開発したターミナルベースの AI コーディングアシスタントです。コードベースの理解、複数ファイルの編集、コマンド実行などの機能を TUI (Terminal User Interface) で提供します。

## インストール

### Linux / macOS

```bash
curl -fsSL https://antigravity.google/cli/install.sh | bash
```

### Windows

- PowerShell

    ```powershell
    irm https://antigravity.google/cli/install.ps1 | iex
    ```

- コマンドプロンプト

    ```cmd
    curl -fsSL https://antigravity.google/cli/install.cmd -o install.cmd && install.cmd && del install.cmd
    ```

## 初期化 (認証)

インストール後、Antigravity CLI を起動すると Google アカウントによる認証が求められます。

```bash
agy
```

- **ローカル環境**: デフォルトブラウザが自動的に開き、Google アカウントでログインします。
- **リモート / SSH 環境**: SSH セッションを検出し、ターミナルに認証 URL が表示されます。ローカルのブラウザでその URL にアクセスして認証を完了します。

### サインアウト

```bash
/logout
```

## 便利な使い方

corrective_plan.md に改修する内容や、改修後どんなテストを実施するのか等、やるべきことを事前に用意しておき、 `--dangerously-skip-permissions` 付きで実行することで、改修途中のファイル追加・削除、コマンド実行の許可を求めずに改修作業を一気通貫で実行します。

意図しない変更をされないために `--sandbox` を指定したり、無限待ちのようになったら、ある程度待って打ち切らせるために `--print-timeout 30m` （この例では30分待ち）を指定します。

```bash
agy -p "$(cat ./corrective_plan.md)" \
    --dangerously-skip-permissions \
    --sandbox \
    --print-timeout 30m \
    --output-format json
```

## 参考サイト

- [https://github.com/google-antigravity/antigravity-cli](https://github.com/google-antigravity/antigravity-cli)
