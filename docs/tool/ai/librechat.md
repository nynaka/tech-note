---
title: LibreChat
description: LibreChatをソースする手順と設定例です
---

LibreChat のインストール
===

ここでは、コンテナ (Docker) は使用せず、GitHubのソースコードから LibreChat をインストールする手順と各種設定例をまとめます。

## インストール手順

### MongoDB のインストール

- Ubuntu Linux

    ```bash
    # MongoDBの公開GPGキーをインポート
    curl -fsSL https://pgp.mongodb.com/server-7.0.asc | \
      sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

    # MongoDBのリポジトリを追加
    echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
      sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

    # パッケージリストの更新とインストール
    sudo apt-get update
    sudo apt-get install -y mongodb-org

    # MongoDBサービスの起動と自動起動の有効化
    sudo systemctl start mongod
    sudo systemctl enable mongod
    ```


### Node.js のインストール

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

- Node.js のインストール (ここでは v24.16.0)

    ```bash
    nvm install v24.16.0
    ```

- バージョンの確認

    ```basb
    node -v
    npm -v
    npx -v
    ```


### LibreChat

- ソースコードの取得

    ```bash
    git clone https://github.com/danny-avila/LibreChat.git
    cd LibreChat
    ```

- LibreChat 依存パッケージをインストールします。

    ```bash
    npm ci
    ```

- フロントエンドのビルド

    Reactベースのフロントエンドの静的ファイルをビルドします。

    ```bash
    npm run frontend
    ```

- 環境変数の設定と初期化

    `.env.example` をコピーして `.env` ファイルを作成します。

    ```bash
    cp .env.example .env
    ```

    - MongoDB 接続設定

        作成した `.env` をエディタで開き、MongoDBの接続文字列を設定します。  
        ローカルで起動した MongoDB を使用する場合、以下のように設定します (通常はデフォルトのままで動作します)。

        ```env title=".env"
        MONGO_URI="mongodb://127.0.0.1:27017/LibreChat"
        ```

        :::note
        LibreChat は初回起動時に指定された MongoDB のデータベース (上記の場合は `LibreChat`) へ自動的に接続し、必要なコレクションやインデックスの初期化を行います。  
        手動でデータベースやテーブルを事前に作成しておく必要はありません。
        :::

    - バインドする IP:port

        ```env title=".env"
        #HOST=localhost
        HOST=0.0.0.0
        PORT=3080
        ```

        デフォルト設定だと `127.0.0.1:3080` をバインドするため、LibreChat を起動したホスト上からでしか接続できません。  
        SSH ポートフォワーディングとか利用すればよいのですが、上記の例では、面倒なので `0.0.0.0:3080` をバインドするようにしています。

- バックエンドの起動

    ビルドと設定が完了したら、バックエンドサーバーを起動します。

    ```bash
    npm run backend
    ```

    起動後、ブラウザから `http://localhost:3080` (デフォルトポートの場合) にアクセスしてください。  
    最初のアクセス時に管理者アカウント (最初のユーザー) の作成画面が表示されますので、画面の指示に従ってユーザー登録を完了させると LibreChat が利用可能になります。

---

## 設定例

LLM (大規模言語モデル) を利用するための設定例です。

### ChatGPT を使用する時の設定例

OpenAIのモデル（ChatGPT等）を使用する場合は、`.env` ファイルにAPIキーを記述するだけで利用可能になります。

```env title=".env"
# OpenAI APIキーを設定します
OPENAI_API_KEY="sk-your-openai-api-key-here"
```

### Gemini を使用する時の設定例

GoogleのGeminiモデルを使用する場合も同様に、`.env` ファイルにAPIキーを記述します。

```env title=".env"
# Google Gemini APIキーを設定します
GOOGLE_KEY="your-google-gemini-api-key-here"
```

### Ollama で Gemma4 を利用する時の設定例

Ollama などローカルで稼働しているカスタムエンドポイントを使用する場合は、プロジェクトのルートディレクトリに `librechat.yaml` という設定ファイルを作成 (または編集) して追加の設定を行います。

※事前にローカルでOllamaを起動し、コマンドプロンプトやターミナルから `ollama pull gemma4` を実行してGemma4のモデルをダウンロードしておいてください。

```yaml title="librechat.yaml"
version: 1.1.5
endpoints:
  custom:
    - name: "Ollama"
      # OllamaではAPIキーは通常無視されますが、設定上何らかの文字列が必要です
      apiKey: "ollama"
      # Dockerを使用しないローカル環境同士の通信なのでlocalhostを指定します
      baseURL: "http://localhost:11434/v1/"
      models:
        default: # ollama list で表示されるモデル名 (`:latest` は除く) と一致させる必要があります
          - "igorls/gemma-4-E4B-it-qat-q4_0-unquantized-heretic"
          - "igorls/gemma-4-12B-it-qat-q4_0-unquantized-heretic"
      modelDisplayLabel: "Ollama (Gemma4 QAT)"
      fetch: false # models.defaultで明示的にモデルを指定しているためfalseにします
```

設定ファイル (`librechat.yaml` や `.env`) を変更した後は、設定を反映させるために `npm run backend` を再起動してください。




### 1. 前提条件
以下のソフトウェアがシステムにインストールされている必要があります。
- **Node.js** (v20.19以上推奨)
- **Git**
- (オプション) MeiliSearch ※チャットの検索機能を利用する場合

