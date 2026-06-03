---
title: MARP
description: Markdown からプレゼン資料を作成するための資料です。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---
# Marp: Markdownからのプレゼン資料作成

## 1. Marp とは

[Marp](https://marp.app/) は Markdown からスライドデッキを作成するためのオープンソースツールエコシステムです。

### 特徴

- **CommonMark ベース** — 通常の Markdown の書き方がそのまま使える。`---` でスライドを区切るだけ
- **多形式エクスポート** — HTML / PDF / PowerPoint (PPTX) への変換をサポート
- **テーマ・カスタマイズ** — CSS によるカスタムテーマが作成可能
- **完全オープンソース** — MIT ライセンス、すべてのツールが公開されている

### エコシステム

| ツール                                                                                        | 説明                             |
| --------------------------------------------------------------------------------------------- | -------------------------------- |
| [Marp Core](https://github.com/marp-team/marp-core)                                           | コアコンバーター。公式テーマ同梱 |
| [Marp CLI](https://github.com/marp-team/marp-cli)                                             | コマンドラインツール             |
| [Marp for VS Code](https://marketplace.visualstudio.com/items?itemName=marp-team.marp-vscode) | VS Code 拡張機能                 |
| [Marpit](https://marpit.marp.app/)                                                            | 基盤フレームワーク（開発者向け） |

---

## 2. Marp CLI のインストールと使い方

### インストール

#### Homebrew（macOS / Linux）

```bash
brew install marp-cli
```

#### Scoop（Windows）

```bash
scoop install marp
```

#### npm（グローバルインストール）

```bash
npm install -g @marp-team/marp-cli
```

#### npm（プロジェクトローカル）

```bash
npm install --save-dev @marp-team/marp-cli
```

#### インストール不要で試す（npx）

Node.js v18 以降があれば、インストールなしで実行できます。

```bash
npx @marp-team/marp-cli@latest slide-deck.md
```

#### Docker

```bash
docker pull marpteam/marp-cli
docker run --rm -v $PWD:/home/marp/app/ marpteam/marp-cli slide-deck.md
```

---

### 基本的な使い方

#### HTML に変換（デフォルト）

```bash
marp slide-deck.md
# → slide-deck.html が生成される

marp slide-deck.md -o output.html
```

#### PDF に変換

> ※ ブラウザ（Chrome / Edge / Firefox）が必要です

```bash
marp --pdf slide-deck.md
marp slide-deck.md -o output.pdf

# PDF にノートを付与
marp --pdf --pdf-notes slide-deck.md

# PDF にしおりを付与
marp --pdf --pdf-outlines slide-deck.md
```

#### PowerPoint に変換

```bash
marp --pptx slide-deck.md
marp slide-deck.md -o output.pptx
```

#### 画像に変換

```bash
# PNG（全スライドを個別ファイルで出力）
marp --images png slide-deck.md

# JPEG
marp --images jpeg slide-deck.md
```

#### ウォッチモード（ファイル変更を監視）

```bash
marp -w slide-deck.md
```

#### サーバーモード（ブラウザでプレビュー）

```bash
marp -s ./slides
# http://localhost:8080 でアクセス可能
```

#### 複数ファイルの一括変換

```bash
# ディレクトリ内の全 .md ファイルを変換
marp --pdf ./slides/

# 入力ディレクトリ構造を保ってまとめて出力
marp --input-dir ./slides/ --output ./output/ --pdf
```

---

### 設定ファイル（`.marprc.yml`）

プロジェクトルートに置くことで設定を共有できます。

```yaml
# .marprc.yml
theme: gaia
size: 16:9
html: true
pdf: true
```

---

## 3. VS Code 拡張機能（Marp for VS Code）

### インストール

VS Code の拡張機能マーケットプレイスで `Marp` と検索するか、以下のコマンドでインストール。

```
ext install marp-team.marp-vscode
```

または [マーケットプレイスページ](https://marketplace.visualstudio.com/items?itemName=marp-team.marp-vscode) から直接インストール。

### 有効化

Front matter に `marp: true` を記述するだけで Marp 機能が有効になります。

```markdown
---
marp: true
---

# スライドタイトル
```

### 主な機能

| 機能                    | 説明                                                           |
| ----------------------- | -------------------------------------------------------------- |
| **プレビュー**          | Markdown の変更をリアルタイムでスライド表示                    |
| **IntelliSense**        | ディレクティブの自動補完・シンタックスハイライト・ホバーヘルプ |
| **診断（Diagnostics）** | ディレクティブの問題を検出してエラー表示                       |
| **エクスポート**        | コマンドパレットから HTML / PDF / PPTX に直接エクスポート      |
| **カスタムテーマ**      | ワークスペース設定でカスタム CSS テーマを参照可能              |

### エクスポート手順

1. コマンドパレット（`Ctrl+Shift+P` / `Cmd+Shift+P`）を開く
2. `Marp: Export slide deck...` を選択
3. 出力形式（HTML / PDF / PPTX / PNG / JPEG）を選択して保存

### カスタムテーマの設定（`.vscode/settings.json`）

```json
{
  "markdown.marp.themes": [
    "./themes/my-theme.css"
  ]
}
```

---

## 4. テンプレートとテーマ

### 基本テンプレート

```markdown
---
marp: true
theme: default
paginate: true
header: "ヘッダーテキスト"
footer: "フッターテキスト"
---

# スライドタイトル

発表者名 / 日付

---

## アジェンダ

1. トピック 1
2. トピック 2
3. トピック 3

---

## コンテンツスライド

- 箇条書き 1
- 箇条書き 2

> 引用・補足情報

---

## まとめ

- ポイント 1
- ポイント 2

---

<!-- _class: lead -->

# ご清聴ありがとうございました
```

---

### 公式テーマ（Marp Core 同梱）

Marp Core には 3 つの公式テーマが含まれています。

#### `default`

シンプルで汎用的な白背景テーマ。

```markdown
---
marp: true
theme: default
---
```

#### `gaia`

青・緑系のカラフルなテーマ。`lead` クラスで表紙スライドに対応。

```markdown
---
marp: true
theme: gaia
---

<!-- _class: lead -->
# タイトルスライド
```

#### `uncover`

洗練されたミニマルデザインのテーマ。

```markdown
---
marp: true
theme: uncover
---
```

---

### テーマのカスタマイズ

#### スライドサイズの変更

```markdown
---
marp: true
theme: gaia
size: 4:3    # 960x720（デフォルトは 16:9 = 1280x720）
---
```

#### インラインスタイルの適用（`<style>` タグ）

```markdown
---
marp: true
theme: default
---

<style>
  section {
    background-color: #f5f5f5;
    font-family: "Noto Sans JP", sans-serif;
  }
  h1 {
    color: #2c3e50;
  }
</style>
```

#### 特定スライドのみスタイル変更（`<style scoped>`）

```markdown
---

<!-- このスライドだけに適用 -->
<style scoped>
  section { background-color: #1a1a2e; color: white; }
</style>

## ダークスライド

このスライドだけ背景が暗くなります。
```

---

### サードパーティ・コミュニティテーマ

| テーマ             | 概要                           | URL                                                      |
| ------------------ | ------------------------------ | -------------------------------------------------------- |
| **marp-academic**  | 学術発表向けクリーンテーマ     | [GitHub](https://github.com/kszk-benedekb/marp-academic) |
| **marp-corporate** | コーポレート向けビジネステーマ | 各種コミュニティ公開あり                                 |
| **beam**           | LaTeX Beamer 風テーマ          | [GitHub 検索](https://github.com/search?q=marp+theme)    |

カスタムテーマは CSS ファイルとして作成し、Front matter で指定します。

```markdown
---
marp: true
theme: my-custom-theme
---
```

```bash
# CLI でカスタムテーマを使用
marp --theme ./themes/my-custom-theme.css slide-deck.md
```

---

## 5. Marp Tips

### 背景画像の設定

```markdown
---

![bg](https://example.com/image.jpg)

## 背景画像付きスライド

---

<!-- 明るさを調整 -->
![bg brightness:0.5](./image.jpg)

## 暗くした背景

---

<!-- サイズ指定 -->
![bg cover](./image.jpg)    <!-- カバー（デフォルト） -->
![bg contain](./image.jpg)  <!-- 全体表示 -->
![bg 50%](./image.jpg)      <!-- サイズ指定 -->
```

### ページ番号・ヘッダー・フッター

```markdown
---
marp: true
paginate: true
header: "My Presentation"
footer: "© 2024 Author"
---
```

特定スライドのみヘッダー・フッターを非表示にする（ローカルディレクティブ）：

```markdown
---
<!-- _header: "" -->
<!-- _footer: "" -->
<!-- _paginate: false -->

## このスライドはヘッダー・フッターなし
```

### スライドクラスの活用

```markdown
---
marp: true
theme: gaia
---

<!-- _class: lead -->
# タイトル（中央寄せ）

---

<!-- _class: invert -->
## 反転カラー（テーマによる）
```

### 数式の表示（KaTeX）

```markdown
インライン数式: $E = mc^2$

ブロック数式:

$$
\sum_{i=1}^{n} x_i = \frac{n(n+1)}{2}
$$
```

### コードブロックのシンタックスハイライト

````markdown
```python
def hello(name: str) -> str:
    return f"Hello, {name}!"
```
````

---

### 左右 2 カラムレイアウト

Marp では CSS の `columns` プロパティや Grid / Flexbox を使って左右に分割したスライドを作れます。

#### 方法 1: `<style scoped>` + `columns` プロパティ（シンプル）

```markdown
---

<style scoped>
section {
  columns: 2;
  column-gap: 2em;
}
h2 {
  column-span: all;  /* 見出しは全幅 */
}
</style>

## 2カラムレイアウト

**左側のコンテンツ**

- 項目 A
- 項目 B
- 項目 C

**右側のコンテンツ**

- 項目 D
- 項目 E
- 項目 F
```

#### 方法 2: HTML + Flexbox（細かい制御に最適）

> Front matter に `html: true` が必要です（または `--html` オプション）

````markdown
---
marp: true
html: true
---

## 2カラム（Flexbox）

<div style="display: flex; gap: 2em;">
<div style="flex: 1;">

**左ブロック**

- 箇条書き 1
- 箇条書き 2
- 箇条書き 3

</div>
<div style="flex: 1;">

**右ブロック**

```python
def greet():
    print("Hello!")
```

</div>
</div>
````

#### 方法 3: 背景画像で左右分割（`bg left` / `bg right`）

画像とテキストを左右に並べたい場合に便利です。

```markdown
---

![bg left:40%](./image.jpg)

## 右側にテキスト

- ポイント 1
- ポイント 2
- ポイント 3

---

![bg right:40%](./image.jpg)

## 左側にテキスト

- ポイント 1
- ポイント 2
```

#### 方法 4: カスタムテーマで再利用可能なクラスを定義

繰り返し使う場合は CSS テーマにクラスを定義しておくのがベストプラクティスです。

```css
/* my-theme.css */
section.cols-2 {
  display: grid;
  grid-template-columns: 1fr 1fr;
  grid-template-rows: auto 1fr;
  gap: 1em;
}

section.cols-2 h2 {
  grid-column: 1 / -1;  /* 見出しは全幅 */
}
```

```markdown
---
marp: true
theme: my-theme
---

<!-- _class: cols-2 -->

## タイトル（全幅）

**左コンテンツ**

- 項目 1
- 項目 2

**右コンテンツ**

- 項目 3
- 項目 4
```

---

### プレゼンターノートの追加

スライドには表示されないノートを HTML コメントとして記述できます。

```markdown
---

## スライドタイトル

ここがスライドの内容

<!-- ここはプレゼンターノート。聴衆には見えない。 -->
<!-- PDF 出力時は --pdf-notes オプションで付与できる -->
```

---

### CI/CD での自動変換（GitHub Actions 例）

```yaml
# .github/workflows/marp.yml
name: Build Slides

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build slides
        uses: docker://marpteam/marp-cli:latest
        with:
          args: --html --pdf slides/deck.md -o output/deck.pdf
        env:
          MARP_USER: root
      - uses: actions/upload-artifact@v4
        with:
          name: slides
          path: output/
```

---

## 参考リンク

- [Marp 公式サイト](https://marp.app/)
- [Marpit フレームワーク ドキュメント](https://marpit.marp.app/)
- [Marp Core GitHub](https://github.com/marp-team/marp-core)
- [Marp CLI GitHub](https://github.com/marp-team/marp-cli)
- [Marp for VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=marp-team.marp-vscode)
- [Marp Core テーマ一覧](https://github.com/marp-team/marp-core/tree/main/themes)
