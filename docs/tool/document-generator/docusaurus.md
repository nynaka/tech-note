---
title: Docusaurus
description: Meta が開発した React ベースの静的サイトジェネレーターのインストールメモです。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

[Docusaurus](https://docusaurus.io/)
===

## 概要

Docusaurus は Meta（旧 Facebook）が開発した、React ベースの静的サイトジェネレーターです。Markdown や MDX でドキュメントを記述でき、バージョン管理・多言語対応・プラグイン拡張など、ドキュメントサイト向けの機能が豊富に揃っています。

## インストール

### Nodejs のインストール

```bash title="Nodejs のインストール"
# nvm のインストール
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
# Nodejs のインストール
source $HOME/.nvm/nvm.sh && nvm install --lts
```

```bash title="環境変数の設定"
source $HOME/.bashrc
```

### Docusaurus のインストール

```bash title="Docusaurus のインストールコマンド"
npx create-docusaurus@latest <プロジェクトフォルダ名> <使用するテンプレート名>
```

プロジェクトフォルダ名に縛りは無いようですが、必ず新規ディレクトリを作成する仕様らしく、カレントディレクトリや、すでに作成されているディレクトリを指定するとエラーになるようです。


## Docusaurus の操作

### 起動

```bash
# npm を使用する場合
npm start -- --host 0.0.0.0 --port 3000
# npx を使用する場合
npx docusaurus start --host 0.0.0.0 --port 3000
```

### エクスポート

```bash
# npm を使用する場合
npm run build -- --out-dir public
# npx を使用する場合
npx docusaurus build --out-dir public
```

## その他

### .gitlab-ci.yml

gitlab-runner を使用している場合、下記のような CI 設定ファイルを用意しておくと、main にプッシュするたびにプロジェクトをビルドしてくれます。

```bash
echo "image: node:24
pages:
  stage: deploy
  script:
    - npm install
    - npm run build -- --out-dir public
  artifacts:
    paths:
      - public
  only:
    - main
" > .gitlab-ci.yml
```
