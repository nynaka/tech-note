---
title: vscode-reveal
description: VS Code の拡張機能 vscode-reveal を使用して、Markdown から Reveal.js ベースのプレゼン資料を作成する手順です。
---

# vscode-reveal で Markdown からプレゼン作成

[vscode-reveal](https://marketplace.visualstudio.com/items?itemName=evilz.vscode-reveal) は、VS Code 上で Markdown を記述し、そのまま [Reveal.js](https://revealjs.com/) ベースのプレゼンテーションとして表示・書き出しができる拡張機能です。

## インストール

1. VS Code の拡張機能マーケットプレイスを開きます (`Ctrl+Shift+X`)。
2. `vscode-reveal` を検索してインストールします（拡張機能 ID: `evilz.vscode-reveal`）。

インストール後、任意の `.md` ファイルを開くとエディタ右上にスライドショーアイコンが表示されます。

## 基本的な使い方

### ファイルの作成

プレゼンテーションは通常の `.md` ファイルです。新規ファイルを作成し、先頭に YAML Front Matter を記述するだけで始められます。

```markdown
---
theme: black
---

# 最初のスライド
内容を書きます
```

### スライドの区切り

Markdown ファイル内で特定の記号を使用することで、スライドを区切ることができます。

- **水平スライド (横移動)**: `---` をその行だけに単独で置く
- **垂直スライド (下移動)**: `--` をその行だけに単独で置く ※デフォルト設定の場合

区切り記号は YAML Front Matter の `separator` / `verticalSeparator` で変更できます。

```yaml
---
separator: "^\n---\n"
verticalSeparator: "^\n--\n"
---
```

:::caution
`---` は Markdown の水平線とも解釈されるため、意図しないスライド分割が起きることがあります。`separator` を正規表現で明示的に指定すると誤動作を防げます。
:::

### プレビューの表示

- エディタの右上にある **「Revealjs: Show presentation by side」** アイコンをクリックします。
- または、コマンドパレット (`Ctrl+Shift+P`) で以下のコマンドを選択します:
  - `Revealjs: Show presentation by side` — VS Code 内サイドパネルで表示
  - `Revealjs: Open presentation in browser` — 既定ブラウザで表示

ブラウザ表示は通常 `http://localhost:3000` で起動します。ポートが使用中の場合は自動的に別ポートが選ばれます。

## 設定 (YAML Front Matter)

Markdown の先頭に YAML 形式で設定を記述することで、プレゼンテーションの挙動や見た目をカスタマイズできます。

```yaml
---
theme: black
transition: slide
highlightTheme: monokai
controls: true
progress: true
slideNumber: true
---
```

### 主な設定項目

| 項目 | 説明 | 値の例 |
|------|------|--------|
| `theme` | スライド全体のテーマ | `black`, `white`, `night` など |
| `transition` | スライド切り替えアニメーション | `none`, `fade`, `slide`, `convex`, `concave`, `zoom` |
| `backgroundTransition` | 背景のトランジション | `none`, `fade`, `slide` など |
| `highlightTheme` | コードハイライトのテーマ | `monokai`, `github`, `vs2015` など |
| `controls` | ナビゲーションコントロールの表示 | `true` / `false` |
| `progress` | 進捗バーの表示 | `true` / `false` |
| `slideNumber` | スライド番号の表示形式 | `true`, `false`, `'c'`, `'c/t'`, `'h/v'`, `'h.v'` |
| `width` | スライドの幅 (px) | `1280` |
| `height` | スライドの高さ (px) | `720` |
| `center` | スライドを垂直方向に中央揃え | `true` / `false` |
| `loop` | プレゼンをループ再生 | `true` / `false` |
| `autoSlide` | 自動スライド送り間隔 (ms、0 で無効) | `3000` |
| `mouseWheel` | マウスホイールでスライド移動 | `true` / `false` |
| `hash` | URL ハッシュにスライド番号を反映 | `true` / `false` |
| `separator` | 水平スライドの区切り (正規表現) | `"^\n---\n"` |
| `verticalSeparator` | 垂直スライドの区切り (正規表現) | `"^\n--\n"` |
| `notesSeparator` | スピーカーノートの区切り | `"note:"` |

## テーマ

### 標準テーマ
`black` (デフォルト), `white`, `league`, `beige`, `sky`, `night`, `serif`, `simple`, `solarized`, `blood`, `moon`

### カスタム CSS
独自のスタイルを適用したい場合は、CSS ファイルを作成して `customTheme` で指定します。パスは Markdown ファイルからの相対パスで、**拡張子 `.css` は不要**です（内部で自動付与されます）。

```yaml
---
customTheme: "my-style"
---
```

カスタム CSS の例:

```css
/* my-style.css */
.reveal h1, .reveal h2 {
  color: #4a90d9;
  font-family: 'Noto Sans JP', sans-serif;
}

.reveal .slides section {
  text-align: left;
}

/* コードブロックのフォントサイズ調整 */
.reveal pre code {
  font-size: 0.8em;
  line-height: 1.5;
}
```

## 小技・テクニック

### スピーカーノート
`note:` 行以降に記述した内容は、プレゼンター画面でのみ表示されるメモになります。プレゼン中に `S` キーを押すとスピーカービューが開きます。

```markdown
# スライドタイトル
スライドの内容

note:
ここには発表時のカンニングペーパーを書きます。
```

> `note:` の区切り文字は YAML Front Matter の `notesSeparator` で変更できます。HTML の `<aside class="notes">...</aside>` タグも使えます。

### フラグメント (段階表示)
リストなどを1つずつ表示させたい場合は、HTML コメントを使用します。

```markdown
- 項目 1 <!-- .element: class="fragment" -->
- 項目 2 <!-- .element: class="fragment fade-up" -->
- 項目 3 <!-- .element: class="fragment highlight-red" -->
```

主なアニメーションクラス: `fade-up`, `fade-down`, `fade-left`, `fade-right`, `fade-out`, `grow`, `shrink`, `highlight-red`, `highlight-green`, `highlight-blue`

表示順を制御したい場合は `data-fragment-index` を使います。

```markdown
- 後から表示 <!-- .element: class="fragment" data-fragment-index="2" -->
- 先に表示   <!-- .element: class="fragment" data-fragment-index="1" -->
```

### 自動アニメーション (Auto-Animate)
同じ `data-id` を持つ要素がスライド間で移動したり形を変えたりする際、滑らかなアニメーションを適用できます。アニメーションを有効にするには、**スライド側**に `data-auto-animate` 属性が必要です。

```markdown
<!-- .slide: data-auto-animate -->
# スライド 1 <!-- .element: data-id="box" -->

---

<!-- .slide: data-auto-animate -->
# スライド 2 <!-- .element: data-id="box" style="color: red; font-size: 2em;" -->
```

### 背景の設定
スライドごとに背景画像や色を指定できます。

```markdown
<!-- .slide: data-background="#ff0000" -->
# 赤い背景のスライド

---

<!-- .slide: data-background="./images/photo.jpg" data-background-size="cover" -->
# 画像背景のスライド
```

主な背景属性:

| 属性 | 説明 |
|------|------|
| `data-background` | 色コード、画像パス、または URL |
| `data-background-size` | `cover`, `contain`, またはサイズ指定 |
| `data-background-position` | 位置 (例: `center`, `top left`) |
| `data-background-repeat` | `no-repeat`, `repeat` など |
| `data-background-opacity` | 透明度 (0〜1) |
| `data-background-video` | 動画ファイルのパスまたは URL |
| `data-background-color` | スライドの背景色 (個別指定) |

### 数式の表示
`katex: true` を設定することで、LaTeX 形式の数式を表示できます。

```yaml
---
katex: true
---
```

```markdown
インライン数式: $E = mc^2$

ブロック数式:
$$
\int_0^\infty e^{-x^2} dx = \frac{\sqrt{\pi}}{2}
$$
```

### コードブロックの行ハイライト
コードブロックの言語指定の後に `[行番号]` を記述すると、特定の行を強調表示できます。`|` 区切りで複数のステップを設定すると、クリックするたびにハイライトが切り替わります。

````markdown
```python [1-3|5-7|9]
# ステップ1: 1〜3行目をハイライト
def setup():
    pass

# ステップ2: 5〜7行目をハイライト
def process():
    return result

run()  # ステップ3: 9行目をハイライト
```
````

### スライドの表示制御 (data-visibility)
特定のスライドをページ番号から除外したり、完全に非表示にしたりできます。

```markdown
<!-- .slide: data-visibility="hidden" -->
# このスライドは表示されない

---

<!-- .slide: data-visibility="uncounted" -->
# 表示されるがページ番号に含まれない（補足スライドに便利）
```

### 画像の挿入
通常の Markdown 記法が使えます。パスは Markdown ファイルからの相対パスです。

```markdown
![説明テキスト](./images/diagram.png)

<!-- サイズを指定したい場合は HTML タグを使用 -->
<img src="./images/diagram.png" width="400" alt="説明テキスト">

<!-- フラグメントとして段階表示 -->
![説明](./images/step1.png) <!-- .element: class="fragment" -->
```

### マルチカラムレイアウト
HTML を直接記述することで2カラムレイアウトなどが実現できます。

```html
<div style="display: flex; gap: 2em;">
  <div style="flex: 1;">

  ### 左カラム
  - 項目 A
  - 項目 B

  </div>
  <div style="flex: 1;">

  ### 右カラム
  - 項目 C
  - 項目 D

  </div>
</div>
```

### プレゼン中のキーボードショートカット

| キー | 動作 |
|------|------|
| `→` / `Space` | 次のスライドへ |
| `←` | 前のスライドへ |
| `↑` / `↓` | 垂直スライドの移動 |
| `F` | フルスクリーン |
| `S` | スピーカービューを開く |
| `Esc` / `O` | スライド一覧 (Overview) |
| `B` または `.` | 画面を暗転 |
| `Alt+Click` | ズームイン/アウト |
| `?` | ショートカット一覧を表示 |

## エクスポート

作成したプレゼンを共有可能な形式で書き出すことができます。

### HTML (静的サイト) として書き出し
コマンドパレットから `Revealjs: Export in HTML` を実行します。Reveal.js のライブラリも同梱された自己完結型の HTML ファイルが生成されます。

### PDF として書き出し
コマンドパレットから `Revealjs: Export in PDF` を実行します。内部的にはブラウザの印刷機能を使用して PDF を生成します。

別の方法として、ブラウザでプレゼンを開いている場合は URL に `?print-pdf` を付加し、ブラウザの印刷ダイアログから「PDF として保存」を選択することでも PDF 化できます。

```
http://localhost:3000/?print-pdf
```

:::tip
PDF 出力時に背景色や画像が消える場合は、ブラウザの印刷設定で **「背景のグラフィックス」を有効** にしてください。
:::

---

## ベーステンプレート

以下の内容を新しい `.md` ファイルにコピーして使い始めてください。

````markdown
---
title: プレゼンテーションのタイトル
theme: night
transition: slide
highlightTheme: monokai
slideNumber: true
---

# プレゼンタイトル
## サブタイトル
作成者：名前

---

# 概要
- vscode-reveal の紹介
- Markdown で書ける
- Reveal.js ベース

---

# 特徴

## 1. 垂直スライド
下方向にもスライドを作れます

--

## 1-1. 補足情報
このように深掘りできます

---

# コードの表示

```python
def hello():
    print("Hello, Reveal.js!")
```

---

# 最後に
ご清聴ありがとうございました
````
