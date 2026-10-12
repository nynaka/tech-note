---
title: Codex CLI
description: OpenAI Codex CLI のインストールに関する手順です。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

[Codex CLI](https://github.com/openai/codex)
===

OpenAI Codex CLI は、ターミナル上で動作する OpenAI 公式の軽量なコーディングエージェントです。
コードベースの解析、コードの生成や編集、コマンド実行などを対話型インターフェースでサポートします。

## 前提条件

- OpenAI アカウント / ChatGPT アカウント
    - **ChatGPT アカウント (推奨)**: ChatGPT Plus、Pro、Team、Enterprise などでログインして利用可能
    - **OpenAI API キー**: API 従量課金で使用する場合に必要 (`OPENAI_API_KEY`)
- Node.js 22 以降 (npm 経由でインストールする場合)
- インターネット接続必須

---

## システム要件

| 項目       | 要件                                                        |
| ---------- | ----------------------------------------------------------- |
| OS         | macOS、Linux (Ubuntu, Debian 等)、Windows (PowerShell, WSL) |
| ランタイム | Node.js v22 以降 (npm 版の場合)                             |
| シェル     | Bash、Zsh、PowerShell 等                                    |

---

## インストール

### npm を使用したインストール (クロスプラットフォーム / 推奨)

前提条件: Node.js 22 以降

<details>
<summary>Node.js をインストールしていない場合</summary>

- nvm のインストール

    ```bash
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
    ```

- `.bashrc` の再読み込み

    ```bash
    source $HOME/.bashrc
    ```

- nvm でインストールできる Node.js のバージョン確認

    ```bash
    nvm ls-remote --lts
    ```

- Node.js のインストール (v22 系推奨)

    ```bash
    nvm install 22
    ```

- バージョンの確認

    ```bash
    node -v
    npm -v
    ```
</details>

- Codex CLI のインストール

    ```bash
    npm install -g @openai/codex
    ```

- バージョン確認

    ```bash
    codex --version
    ```

- アップデート

    ```bash
    npm install -g @openai/codex@latest
    ```

---

### Homebrew を使用したインストール (macOS)

- インストール

    ```bash
    brew install codex
    ```

- バージョン確認

    ```bash
    codex --version
    ```

- アップデート

    ```bash
    brew upgrade codex
    ```

---

### インストールスクリプトを使用したインストール

#### Linux / macOS / WSL

- インストール

    ```bash
    curl -fsSL https://chatgpt.com/codex/install.sh | sh
    ```

- バージョン確認

    ```bash
    codex --version
    ```

#### Windows (PowerShell)

- インストール

    ```powershell
    powershell -ExecutionPolicy ByPass -c "irm https://chatgpt.com/codex/install.ps1 | iex"
    ```

- バージョン確認

    ```powershell
    codex --version
    ```

---

## 認証 (セットアップ)

Codex CLI を初回起動すると、認証方法の選択またはログインが求められます。

```bash
codex
```

### 1. ChatGPT アカウントでの認証 (推奨)

画面の指示に従って Enter キーを押すとブラウザが起動し、ChatGPT アカウントへのログイン・承認画面が表示されます。
認証が完了すると、ChatGPT Plus / Pro / Team などのプランで CLI を利用できるようになります。

### 2. OpenAI API キーでの認証

API の従量課金を利用する場合は、環境変数 `OPENAI_API_KEY` を設定してから起動します。

- **Linux / macOS**:

    ```bash
    export OPENAI_API_KEY="sk-..."
    ```

    永続化する場合は `~/.bashrc` や `~/.zshrc` に記載します。

- **Windows (PowerShell)**:

    ```powershell
    $env:OPENAI_API_KEY = "sk-..."
    ```

---

## 基本的な使い方

### 対話型セッションの開始

作業対象のプロジェクトルートディレクトリで `codex` を実行します。

```bash
cd /path/to/project
codex
```

```text
> どのようなお手伝いをしましょうか？
```

ターミナル上で自然言語を用いてコードの作成、不具合の調査、リファクタリングなどを指示できます。

### 主なスラッシュコマンド

| コマンド          | 説明                                    |
| ----------------- | --------------------------------------- |
| `/new`            | 新規セッションを開始                    |
| `/resume`         | 以前のセッションを再開                  |
| `/model`          | 利用するモデルの変更                    |
| `/mode`           | 実行モードの変更 (Read Only / Agent 等) |
| `/status`         | 現在の設定や接続状況を確認              |
| `/quit` / `/exit` | セッションを終了                        |

### 非対話型実行 (スクリプト / CI)

スクリプトや CI/CD で自動実行したい場合は `exec` サブコマンドを利用できます。

```bash
codex exec "テストを実行して失敗した原因を教えてください"
```

### プロジェクト設定ファイル (`AGENTS.md`)

プロジェクトのルートディレクトリに `AGENTS.md` を作成しておくと、Codex CLI が起動時にプロジェクトのアーキテクチャ規約やコーディングスタイル、テストコマンドを読み取って作業に反映します。

---

## アンインストール

### npm でインストールした場合

```bash
npm uninstall -g @openai/codex
```

### Homebrew でインストールした場合

```bash
brew uninstall codex
```

### インストールスクリプトでインストールした場合

- **Linux / macOS**:

    ```bash
    rm -f ~/.local/bin/codex
    rm -rf ~/.codex
    ```

- **Windows (PowerShell)**:

    ```powershell
    Remove-Item -Recurse -Force "$env:USERPROFILE\.codex"
    ```

---

## 参考サイト

- [OpenAI Codex CLI (GitHub)](https://github.com/openai/codex)
- [OpenAI 公式サイト](https://openai.com/)
