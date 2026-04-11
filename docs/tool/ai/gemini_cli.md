---
title: Gemini CLI
description: Gemini CLI のインストールに関する手順です。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

[Gemini CLI](https://cloud.google.com/blog/ja/topics/developers-practitioners/introducing-gemini-cli/)
===

## [インストール](https://github.com/google-gemini/gemini-cli?tab=readme-ov-file#-installation)

### Node.js のインストール

<details>
<summary>Node.js をインストールしていない場合</summary>
- nvm のインストール

    ```bash
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
    ```

- .bashrc の再読み込み

    ```bash
    source $HOME/.bashrc
    ```

- nvm でインストールできる Node.js のバージョン確認

    ```bash
    nvm ls-remote --lts
    ```

- Node.js のインストール (ここでは v24.12.0)

    ```bash
    nvm install v24.13.0
    ```

- バージョンの確認

    ```basb
    node -v
    npm -v
    npx -v
    ```
</details>

### [Gemini CLI](https://github.com/google-gemini/gemini-cli?tab=readme-ov-file#-installation)

- インストール

    ```bash
    npm install -g @google/gemini-cli
    ```

- インストールされたバージョンの確認

    ```bash
    gemini --version
    ```

### 初回実行

```bash
gemini
```

```text
lqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqk
x                                                                                                                      x
x ? Get started                                                                                                        x
x                                                                                                                      x
x   How would you like to authenticate for this project?                                                               x
x                                                                                                                      x
x   ● 1. Login with Google                                                                                             x
x     2. Use Gemini API Key                                                                                            x
x     3. Vertex AI                                                                                                     x
x                                                                                                                      x
x   No authentication method selected.                                                                                 x
x                                                                                                                      x
x   (Use Enter to select)                                                                                              x
x                                                                                                                      x
x   Terms of Services and Privacy Notice for Gemini CLI                                                                x
x                                                                                                                      x
x   https://github.com/google-gemini/gemini-cli/blob/main/docs/tos-privacy.md                                          x
x                                                                                                                      x
mqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqj
```

提示された URL を参照し、認証後に表示される認証コードをコピペすると認証処理が実行され、Gemini CLI を使用できるようになります。


### [Gemini Code Assist](https://marketplace.visualstudio.com/items?itemName=Google.geminicodeassist)

Gemini CLI をインストールした Linux 環境に VSCode でログインし、Gemini Code Assist をインストールすると、Gemini CLI を VSCode から使用できるようになり、Github Copilot のような使い方ができるようになります。


## 参考サイト

- [https://codelabs.developers.google.com/gemini-cli-hands-on?hl=ja#0](https://codelabs.developers.google.com/gemini-cli-hands-on?hl=ja#0)