---
title: Claude Code
description: Claude Code のインストールに関する手順です。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

[Claude Code](https://code.claude.com/docs/ja/overview)
===

## 前提条件

- 有効な Anthropic アカウント (Pro / Max / Team / Enterprise / Console)
    - 無料の Claude.ai プランは Claude Code に対応していません
    - [Anthropic プランの確認](https://www.anthropic.com/pricing)
- Windows の場合、[Git for Windows](https://git-scm.com/downloads/win) のインストールが必要

---

## システム要件

| 項目 | 要件 |
|------|------|
| OS | macOS 13.0+、Windows 10 1809+ / Server 2019+、Ubuntu 20.04+、Debian 10+、Alpine Linux 3.19+ |
| ハードウェア | RAM 4 GB 以上、x64 または ARM64 プロセッサ |
| ネットワーク | インターネット接続必須 |
| シェル | Bash、Zsh、PowerShell、CMD |

---

## Linux / macOS / WSL

### インストールスクリプトを使用したインストール (推奨)

- インストール

    ```bash
    curl -fsSL https://claude.ai/install.sh | bash
    ```

- バージョン確認

    ```bash
    claude --version
    ```

### Homebrew を使用したインストール (macOS)

- 安定版のインストール

    ```bash
    brew install --cask claude-code
    ```

- 最新版のインストール

    ```bash
    brew install --cask claude-code@latest
    ```

- バージョン確認

    ```bash
    claude --version
    ```

- アップデート

    ```bash
    # 安定版の場合
    brew upgrade claude-code
    # 最新版の場合
    brew upgrade claude-code@latest
    ```

---

## Windows

### WinGet を使用したインストール (推奨)

前提条件: [Git for Windows](https://git-scm.com/downloads/win) のインストール

- インストール

    ```powershell
    winget install Anthropic.ClaudeCode
    ```

- バージョン確認

    ```powershell
    claude --version
    ```

### インストールスクリプトを使用したインストール

- PowerShell

    ```powershell
    irm https://claude.ai/install.ps1 | iex
    ```

- CMD

    ```cmd
    curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
    ```

---

## npm を使用したインストール (非推奨)

> ネイティブインストーラーへの移行を推奨します。npm 版は非推奨です。

前提条件: Node.js 18 以降

- インストール

    ```bash
    npm install -g @anthropic-ai/claude-code
    ```

    :::caution
    `sudo npm install -g` は権限問題やセキュリティリスクの原因となるため使用しないでください。
    :::

- バージョン確認

    ```bash
    claude --version
    ```

---

## 認証

- Claude Code の起動

    ```bash
    claude
    ```

    初回起動時にブラウザが開き、Anthropic アカウントへのログインが求められます。
    画面の指示に従って認証を完了してください。

---

## 使用例

```bash
claude
```

```text
> どのようなお手伝いができますか？
```

ターミナル上で自然言語によってコードの生成・編集・デバッグなどの支援を受けることができます。

---

## アンインストール

### ネイティブインストール (Linux / macOS / WSL)

```bash
rm -f ~/.local/bin/claude
rm -rf ~/.local/share/claude
```

### Homebrew (macOS)

```bash
# 安定版の場合
brew uninstall --cask claude-code
# 最新版の場合
brew uninstall --cask claude-code@latest
```

### WinGet (Windows)

```powershell
winget uninstall Anthropic.ClaudeCode
```

### npm

```bash
npm uninstall -g @anthropic-ai/claude-code
```
