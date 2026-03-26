---
title: MkDocs
description: Python 製の Markdown ベース静的サイトジェネレーター。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

[MkDocs](https://www.mkdocs.org/)
===

## 概要

MkDocs は Python 製の静的サイトジェネレーターです。Markdown でドキュメントを記述し、YAML ファイル一つで設定が完結するシンプルさが特徴です。テーマとして [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) が広く使われており、見やすいデザインのドキュメントサイトを手軽に作成できます。

## インストール

```bash title="requirements.txt"
echo "mkdocs
mkdocs-material
mkdocs-git-revision-date-localized-plugin" \
> requirements.txt
```

```bash title="MkDocs インストール"
python3 -m venv venv
source ./venv/bin/activate
pip install -U -r requirements.txt
```

## プロジェクトの作成

```bash title="プロジェクト作成コマンド"
mkdocs new <プロジェクトフォルダパス>
```

カレントディレクトリをプロジェクトフォルダとする場合は下記のように実行します。

```bash title="カレントディレクトリにプロジェクトを作成する"
mkdocs new .
```

プロジェクト作成後、下記のコマンドを実行し、ブラウザで **http://127.0.0.1:8000/** を参照すると、MkDocs のデフォルト画面を参照することができます。

```bash title="mkdocs 内蔵 Webサーバの起動"
mkdocs serve
```

## mkdocs.yml 設定例

ここでは下記の設定をしています。

- **Home** のラベルで index.md を表示する。
- テーマは **material** を使用し、コードブロックにコピーボタンを表示する。
- **plugins** として以下を有効にする。
    - tags : ページにタグを付けて分類できるようにする。
    - search : 日本語対応の全文検索機能を有効にする。
    - git-revision-date-localized : Gitのコミット日時をページに表示する（MKDOCS_CI 環境変数が true の場合のみ動作）。
- **markdown_extensions** として以下を有効にする。
    - toc : 目次を自動生成する（4階層まで対応）。
    - admonition : Note・Warning などの注意書きブロックを使用できるようにする。
    - pymdownx.details : クリックで展開できる折りたたみブロックを使用できるようにする。
    - footnotes : 脚注を使用できるようにする。
    - pymdownx.highlight : コードブロックのシンタックスハイライトを有効にする。
    - pymdownx.snippets : 外部ファイルの内容をスニペットとして埋め込めるようにする。
    - pymdownx.superfences : コードブロックの入れ子や Mermaid ダイアグラムを使用できるようにする。
    - attr_list : Markdown要素に HTML属性（class や id 等）を付与できるようにする。
    - md_in_html : HTML ブロック内で Markdown を記述できるようにする。

```diff title="mkdocs.yml 設定例"
--- mkdocs.yml.origin   2026-03-25 08:58:04.357044593 +0900
+++ mkdocs.yml  2026-03-25 09:01:56.587184376 +0900
@@ -1 +1,63 @@
 site_name: My Docs
+
+nav:
+  - Home: index.md
+
+theme:
+  name: material
+  lang: ja
+  palette:
+    # Palette toggle for light mode
+    - media: "(prefers-color-scheme: light)"
+      primary: blue
+      scheme: default
+      toggle:
+        icon: material/brightness-7
+        name: Switch to dark mode
+
+    # Palette toggle for dark mode
+    - media: "(prefers-color-scheme: dark)"
+      primary: blue grey
+      scheme: slate
+      toggle:
+        icon: material/brightness-4
+        name: Switch to light mode
+  features:
+    # ナビゲーションのグループ化表示制御
+    #- navigation.sections
+    # コードブロックのコピーボタン
+    - content.code.copy
+
+# フォントサイズ等の画面表示を変更する場合の設定項目
+#extra_css:
+#  - css/custom.css
+
+plugins:
+  - tags
+  - search:
+      lang: ja
+  # pip install mkdocs-git-revision-date-localized-plugin
+  - git-revision-date-localized:
+      # MKDOCS_CI 環境変数が true にセットされていた場合にプラグインが有効
+      enabled: !ENV [MKDOCS_CI, false]
+      enable_creation_date: true
+
+markdown_extensions:
+  - toc:
+      toc_depth: 4
+  - admonition
+  - pymdownx.details
+  - footnotes
+  - pymdownx.highlight:
+      anchor_linenums: true
+      line_spans: __span
+      pygments_lang_class: true
+  - pymdownx.snippets
+  - pymdownx.superfences:
+      custom_fences:
+        - name: mermaid
+          class: mermaid
+          format: !!python/name:pymdownx.superfences.fence_code_format
+  - attr_list
+  - md_in_html
```

---

## その他の設定

### 起動スクリプト

**livereload** を確実に有効にしたい場合は下記のようなスクリプトを用意しておくと便利です。

```bash title="起動スクリプト"
echo 'exec mkdocs serve --livereload "$@"' > mkdocs_serve
chmod u+x mkdocs_serve
```

### ローカルホスト (127.0.0.1) 以外からも接続を受け付ける場合

下記のように **-a IPアドレス:port** を指定すると、指定した IP アドレス、ポート番号で接続待ちをするようになります。
```bash
mkdocs serve -a 0.0.0.0:8000
```

### .gitlab-ci.yml

gitlab-runner を使用している場合、下記のような CI 設定ファイルを用意しておくと、main にプッシュするたびにプロジェクトをビルドしてくれます。

```bash title=".gitlab-ci.yml"
echo "image: python:3.12
pages:
  stage: deploy
  script:
    - pip3 install -r requirements.txt
    - mkdocs build -d public
  artifacts:
    paths:
      - public
  only:
    - main
" > .gitlab-ci.yml
```

### .gitignore

```bash title=".gitignore"
echo "public/
venv/
" > .gitignore
```


## 参考サイト

‐[Reference - Material for MkDocs](https://squidfunk.github.io/mkdocs-material/reference/)
