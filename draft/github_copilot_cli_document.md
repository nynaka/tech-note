# GitHub Copilot CLI を使った技術情報管理

ここでは、GitHub Copilot CLI を使用して Markdown 形式で技術情報を作成・管理するための初期設定および運用手順をまとめます。

> **注意**: GitHub Copilot CLI は現在も急速に開発が進んでいます。本文書の情報が古くなっている場合は、セッション内で `/help` を実行して最新情報を確認してください。

---

## 0. インストールと初期認証

```bash
# npm でインストール（Homebrew や WinGet でも可）
npm install -g @github/copilot

# 認証（ブラウザフローが起動する）
copilot login
```

認証トークンは以下の優先順位で読み込まれます（CI 環境など headless 用途向け）。

```
COPILOT_GITHUB_TOKEN > GH_TOKEN > GITHUB_TOKEN
```

> **補足**: Classic PAT（`ghp_` で始まるもの）は**非対応**です。"Copilot Requests" 権限を持つ fine-grained PAT（v2）を使用してください。

---

## 1. カスタム指示

GitHub Copilot CLI は、複数の場所に配置したカスタム指示ファイルを自動的に読み込み、**すべてを結合して**エージェントに渡します。

### 指示ファイルの種類と配置場所

| ファイル | 用途 |
| --- | --- |
| `~/.copilot/copilot-instructions.md` | 全プロジェクト共通のルール（ユーザーレベル） |
| `.github/copilot-instructions.md` | リポジトリ全体に適用されるルール |
| `.github/instructions/**/*.instructions.md` | 特定のパスやファイル種別に適用される指示 |
| `AGENTS.md`（git root または cwd） | 各種 AI エージェントが参照する指示 |
| `CLAUDE.md` / `GEMINI.md`（リポジトリルート） | 他ツールとも共有できる指示 |

> **補足**: 従来は優先度に基づくフォールバックでしたが、現在は**すべてのカスタム指示ファイルが結合**されてエージェントに渡されます。リポジトリ指示がユーザー指示より常に上書きされるわけではなく、両方が適用されます。

### リポジトリ全体の指示 (`.github/copilot-instructions.md`)

- **役割**: 全セッションで参照される「共通コンテキスト」。
- **記述例**:

    ```markdown
    - 技術記事は docs/ ディレクトリ配下に適切なカテゴリ分けをして保存すること。
    - 画像は static/img/ に配置し、相対パスで参照すること。
    - 図を記述する場合は mermaid 形式とする。
    - 日本語の文章は丁寧語（です・ます）を基本とする。
    ```

- `/init` コマンドを使うと、現在のリポジトリ構造を元にした `copilot-instructions.md` の雛形を自動生成できます。

    ```text
    /init
    ```

    > **補足**: `/init` はリポジトリのコードベースを分析し、ビルド・テスト・リントコマンドや高レベルのアーキテクチャ情報を含む `.github/copilot-instructions.md` を生成または更新します。ファイルが既に存在する場合は改善案を提示し、適用するか選択できます。ファイルを作成したくない場合は `/init suppress` で起動時の通知を非表示にできます。

### パス指定の指示 (`.github/instructions/*.instructions.md`)

特定のディレクトリやファイル種別に絞って指示を適用できます。`applyTo` フィールドで対象を glob 指定します。

```markdown
---
applyTo: "docs/**/*.md"
---
- Docusaurus の Frontmatter (title, description) を必ず設定すること。
- コードブロックには言語指定を付けること。
- 手順書には前提条件セクションを設けること。
```

### 指示の確認と管理

```text
/instructions      # 読み込まれているカスタム指示ファイルの一覧と有効/無効の切り替え
/env               # 読み込み済みの指示・MCP・スキルなど環境情報を表示
```

---

## 2. スキル (Skill) の作成と配置

スキルは、特定のタスク（例: ドキュメントレビュー、コード変換）をエージェントに実行させるための「専門知識パッケージ」です。タスクに一致したときにのみ読み込まれるため、コンテキストウィンドウを効率的に使えます。

### 配置場所と優先順位

スキルは以下の順序で検索されます（前のものが優先されます）。

```
.github/skills/ → .agents/skills/ → ~/.copilot/skills/ → プラグインディレクトリ → COPILOT_SKILLS_DIRS
```

- **プロジェクト固有**: `.github/skills/<skill-name>/SKILL.md`（バージョン管理でチーム共有）
- **ユーザー共通**: `~/.copilot/skills/<skill-name>/SKILL.md`（全プロジェクトで利用）

### `SKILL.md` の作成例

```markdown
---
name: doc-reviewer
description: Docusaurus の Markdown ドキュメントを技術的正確性・構成・表記の観点でレビューするスキル。ドキュメントレビューを依頼されたときに使用する。
---

# ドキュメントレビュー・プロトコル

以下のチェックリストに従ってレビューを実施します。

- [ ] Docusaurus の Frontmatter (title, description) が設定されているか。
- [ ] コードブロックに言語指定があるか。
- [ ] 手順書に前提条件セクションがあるか。
- [ ] 日本語の表記揺れがないか（例: 「サーバ」/「サーバー」）。
```

### スキルの管理コマンド

```text
/skills list                  # 利用可能なスキルの一覧
/skills info <skill-name>     # スキルの詳細と配置場所の確認
/skills reload                # セッション中にスキルを追加した後の再読み込み
/skills                       # スペースバーでスキルの有効/無効を切り替え
```

### スキルの呼び出し

プロンプトでスキル名を指定して明示的に呼び出せます。

```text
> /doc-reviewer スキルを使って @docs/linux/btrfs.md をレビューして
```

---

## 3. MCP サーバーの設定

GitHub MCP サーバーはデフォルトで組み込まれており、GitHub.com 上のリポジトリ・PR・Issue などを操作できます。追加の MCP サーバーは対話的に登録できます。

### MCP サーバーの追加 (`/mcp add`)

セッション内で `/mcp add` を実行するとフォーム形式で設定を入力できます。`Tab` キーでフィールド間を移動し、`Ctrl+S` で保存します。

```text
/mcp add
```

設定は `~/.copilot/mcp-config.json` に保存されます（`COPILOT_HOME` 環境変数で変更可能）。

### 設定ファイルを直接編集する場合

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/your/docs"]
    }
  }
}
```

> **補足**: MCP 設定はデフォルトではユーザーレベル（`~/.copilot/mcp-config.json`）に保存されます。セッション単位の追加設定は `--additional-mcp-config` フラグでも渡せます。また、OAuth 認証が必要なリモート MCP サーバーのトークンが期限切れになった場合は `/mcp auth <server-name>` で再認証できます。

---

## 4. 既存 Markdown からのチェックリスト作成手順

既存のドキュメントから「レビュー用チェックリスト」を抽出し、ナレッジの品質を統一する手順です。

### 手順 A: Plan モードによる安全な調査・抽出

`Shift+Tab` で Plan モード（読み取り専用）に切り替えてから分析すると、誤ってファイルを変更するリスクを排除できます。

1. **セッション開始**: `copilot` を起動します。
2. **Plan モードに切り替え**: `Shift+Tab` を押してモードを `Plan` に切り替えます。
3. **調査**: 以下のように入力します。

    ```text
    > docs/linux/ のドキュメントを読み込んで、共通する構成ルールをチェックリスト形式で抽出して
    ```

4. **反映**: Plan モードを終了し、抽出されたルールを `.github/copilot-instructions.md` または `.github/skills/doc-reviewer/SKILL.md` に保存するよう指示します。

> **補足**: `Shift+Tab` を押すたびにモードが切り替わります（Default → Plan → Autopilot）。Autopilot モードに入ると、エージェントが承認なしでタスクを自律的に進めるため注意してください。

### 手順 B: `/review` による品質チェック

```text
> /review
```

現在のディレクトリの変更差分をコードレビューエージェントが分析し、問題点を報告します。

### 手順 C: `@` ファイル指定によるピンポイント分析

```text
> @docs/linux/btrfs.md の構成ルールを分析して、チェックリスト形式で書き出して
```

`@` の後にファイルパスを入力するとそのファイルをプロンプトのコンテキストに追加できます。入力中にタブ補完も利用できます。

---

## 5. 推奨ワークフロー

### 対話セッションでの作業（推奨）

```bash
cd /path/to/tech-note
copilot
```

セッション内で自然言語で指示します。

```text
> @docs/linux/ の記事を参考に、Ubuntu 24.04 のセットアップ手順を docs/linux/os/ubuntu_2404.md として作成して
> 作成したファイルを /doc-reviewer スキルを使ってチェックし、修正案を提示して
> 全て適用して
> /diff   ← 変更内容を確認
```

変更を元に戻したい場合は `/rewind` または `/undo` で直前のターンをリバートできます。

### 非対話モード（スクリプト・ワンライナー）

`-p` フラグで非対話的に実行できます。

```bash
copilot -p "docs/linux/ 配下の全ファイルに Frontmatter の title が設定されているか確認して、不足しているファイルを一覧表示して"
```

### 並列実行 (`/fleet`)

大きなタスクを並列サブエージェントに分割して高速化できます。

```text
> /fleet docs/ 配下の全 Markdown ファイルに Frontmatter の title と description を追加して
```

---

## 6. セッション管理と履歴

Copilot CLI はすべてのセッションをローカルに記録します。過去のセッションを再開したり、セッションに関するインサイトを得ることができます。

```text
/session              # セッション一覧の表示・再開
/chronicle            # セッション履歴に基づくインサイトの表示
/context              # コンテキストウィンドウのトークン使用量の内訳を表示
/compact              # 会話履歴を手動で要約してコンテキストを節約
```

> **補足**: コンテキストが上限の 95% に達すると**自動圧縮**が行われます（作業を中断せずにバックグラウンドで処理されます）。

---

## 7. 主要コマンド早見表

### スラッシュコマンド

| コマンド | 動作 |
| --- | --- |
| `/init` | リポジトリ構造を元に `copilot-instructions.md` を自動生成・更新 |
| `/plan` | 実装プランを先に作成してからコーディング |
| `/review` | コードレビューエージェントを起動 |
| `/research` | Web や GitHub を使った深掘り調査 |
| `/diff` | 現在の変更差分を表示 |
| `/rewind` / `/undo` | 直前のターンを取り消してファイル変更を元に戻す |
| `/compact` | 会話履歴を要約してコンテキストを節約 |
| `/share` | セッションを Markdown / HTML / Gist に出力 |
| `/model` | 使用する AI モデルを切り替え |
| `/fleet` | 並列サブエージェントでタスクを分割実行 |
| `/delegate` | タスクを Copilot コーディングエージェントに非同期で委譲（新ブランチ・PR を自動作成） |
| `/agent` | カスタムエージェントを明示的に呼び出す |
| `/session` | セッション一覧の表示・再開・削除 |
| `/chronicle` | セッション履歴のインサイトを表示 |
| `/context` | トークン使用量の内訳を表示 |
| `/mcp` | MCP サーバーの表示・追加・編集・削除・有効化・無効化 |
| `/instructions` | カスタム指示ファイルの一覧と有効/無効の切り替え |
| `/skills` | スキルの一覧と有効/無効の切り替え |
| `/env` | 読み込み済みの指示・MCP・スキルなど環境情報を表示 |
| `/lsp` | 設定済み LSP サーバーの確認 |
| `/feedback` | フィードバック送信・バグ報告・機能要望 |
| `/help` | 利用可能なコマンドの最新一覧を表示 |
| `/clear` | 会話履歴をリセット（セッション承認もリセット） |
| `/exit` / `/quit` | セッションを終了 |

### プロンプト内の特殊記法

| 入力 | 動作 |
| --- | --- |
| `@path/to/file` | ファイルをコンテキストに追加 |
| `#issue-number` | Issue / PR をコンテキストに追加 |
| `!command` | モデルを介さずシェルコマンドを直接実行 |

### キーボードショートカット

| キー | 動作 |
| --- | --- |
| `Shift+Tab` | モード切り替え（Default → Plan → Autopilot） |
| `Esc` | 処理中のタスクを中断（2 回押しで確実にキャンセル） |
| `Ctrl+Y` または `Tab` | 補完候補（`@` メンションなど）を確定 |
| `Ctrl+G` | 現在のプロンプトを外部エディタで編集 |
| `Ctrl+O` | 直近のタイムラインアイテムを展開 |
| `Ctrl+E` | 全タイムラインアイテムを展開 |
| `Ctrl+T` | 推論（thinking）表示の切り替え |

---

## 8. 環境変数と設定ファイル

| 環境変数 | 説明 |
| --- | --- |
| `COPILOT_GITHUB_TOKEN` | 認証トークン（最優先） |
| `COPILOT_HOME` | 設定・状態ファイルの保存先を変更（デフォルト: `~/.copilot/`） |
| `COPILOT_MODEL` | 全セッションで使用するデフォルトモデルを上書き |
| `COPILOT_SKILLS_DIRS` | 追加スキルディレクトリをカンマ区切りで指定 |
| `COPILOT_CUSTOM_INSTRUCTIONS_DIRS` | `AGENTS.md` を追加検索するディレクトリ |
| `COPILOT_AUTO_UPDATE` | `false` に設定すると自動更新をオフにする |

設定ファイルは `~/.copilot/settings.json` に保存されます（内部状態は `config.json` で分離）。

---

## 9. セキュリティ上の注意

- ファイルの変更やコマンドの実行は**すべて事前に承認が必要**です（`--allow-all-tools` を指定しない限り）。
- `--allow-all-tools` を使用する場合、Copilot はあなたと同じファイルアクセス権・シェル実行権限を持ちます。CI 環境や共有環境では使用しないでください。
- デフォルトで SSRF 対策が有効（ループバック・プライベート・リンクローカル・クラウドメタデータアドレスへの接続をブロック）。
- `--show-secrets` フラグは環境変数やヘッダーの機密値をターミナルに出力します。信頼できる環境でのみ使用し、ログや履歴に残さないよう注意してください。