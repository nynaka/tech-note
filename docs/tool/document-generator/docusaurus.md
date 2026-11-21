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

- Nodejs のインストール

    ```bash
    # nvm のインストール
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
    # Nodejs のインストール
    source $HOME/.nvm/nvm.sh && nvm install --lts
    ```

- 環境変数の反映

    ```bash
    source $HOME/.bashrc
    ```

### Docusaurus のインストール

- Docusaurus のインストールコマンド

    ```bash
    npx create-docusaurus@latest <プロジェクトフォルダ名> <使用するテンプレート名>
    ```

    プロジェクトフォルダ名に縛りは無いようですが、必ず新規ディレクトリを作成する仕様らしく、カレントディレクトリや、すでに作成されているディレクトリを指定するとエラーになるようです。

- テンプレート

    - [公式](https://docusaurus.io/docs/api/themes/@docusaurus/theme-classic) では4つ紹介されています。
    - [Styling and Layout](https://docusaurus.io/docs/styling-layout) でカスタマイズする手順が紹介されています。

- 実行例

    ```bash
    npx create-docusaurus@latest sample-site classic
    ```

### 使用しない機能の削除

- デフォルトインストールファイルの削除

    ```text
    sample-site
    + blog ・・・ 削除
    + docs
        + 全部削除して index.md を追加
    + src
        + page ・・・ 削除
    ```

- docusaurus.config.ts の変更差分

    ```diff
    --- /tmp/docusaurus.config.ts   2026-11-21 12:56:47.176906092 +0900
    +++ docusaurus.config.ts        2026-11-21 13:09:17.470117021 +0900
    @@ -22,8 +22,8 @@
    
       // GitHub pages deployment config.
       // If you aren't using GitHub pages, you don't need these.
    -  organizationName: 'facebook', // Usually your GitHub org/user name.
    -  projectName: 'docusaurus', // Usually your repo name.
    +  //organizationName: 'facebook', // Usually your GitHub org/user name.
    +  //projectName: 'docusaurus', // Usually your repo name.
    
       onBrokenLinks: 'throw',
    
    @@ -40,26 +40,8 @@
           'classic',
           {
             docs: {
    +          routeBasePath: "/",
               sidebarPath: './sidebars.ts',
    -          // Please change this to your repo.
    -          // Remove this to remove the "edit this page" links.
    -          editUrl:
    -            'https://github.com/facebook/docusaurus/tree/main/packages/create-docusaurus/templates/shared/',
    -        },
    -        blog: {
    -          showReadingTime: true,
    -          feedOptions: {
    -            type: ['rss', 'atom'],
    -            xslt: true,
    -          },
    -          // Please change this to your repo.
    -          // Remove this to remove the "edit this page" links.
    -          editUrl:
    -            'https://github.com/facebook/docusaurus/tree/main/packages/create-docusaurus/templates/shared/',
    -          // Useful options to enforce blogging best practices
    -          onInlineTags: 'warn',
    -          onInlineAuthors: 'warn',
    -          onUntruncatedBlogPosts: 'warn',
             },
             theme: {
               customCss: './src/css/custom.css',
    @@ -81,65 +63,14 @@
             src: 'img/logo.svg',
           },
           items: [
    -        {
    -          type: 'docSidebar',
    -          sidebarId: 'tutorialSidebar',
    -          position: 'left',
    -          label: 'Tutorial',
    -        },
    -        {to: '/blog', label: 'Blog', position: 'left'},
    -        {
    -          href: 'https://github.com/facebook/docusaurus',
    -          label: 'GitHub',
    -          position: 'right',
    -        },
    -      ],
    -    },
    -    footer: {
    -      style: 'dark',
    -      links: [
    -        {
    -          title: 'Docs',
    -          items: [
    -            {
    -              label: 'Tutorial',
    -              to: '/docs/intro',
    -            },
    -          ],
    -        },
    -        {
    -          title: 'Community',
    -          items: [
    -            {
    -              label: 'Stack Overflow',
    -              href: 'https://stackoverflow.com/questions/tagged/docusaurus',
    -            },
    -            {
    -              label: 'Discord',
    -              href: 'https://discordapp.com/invite/docusaurus',
    -            },
    -            {
    -              label: 'X',
    -              href: 'https://x.com/docusaurus',
    -            },
    -          ],
    -        },
    -        {
    -          title: 'More',
    -          items: [
    -            {
    -              label: 'Blog',
    -              to: '/blog',
    -            },
    -            {
    -              label: 'GitHub',
    -              href: 'https://github.com/facebook/docusaurus',
    -            },
    -          ],
    -        },
           ],
    -      copyright: `Copyright © ${new Date().getFullYear()} My Project, Inc. Built with Docusaurus.`,
         },
    +    //footer: {
    +    //  style: 'dark',
    +    //  links: [
    +    //  ],
    +    //  //copyright: `Copyright © ${new Date().getFullYear()} My Project, Inc. Built with Docusaurus.`,
    +    //},
         prism: {
           theme: prismThemes.github,
           darkTheme: prismThemes.dracula,
    ```

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

### github から clone して起動するまで

```bash
git clone repository.git
npm install    # これやらないとエラーになる
npm start
```
